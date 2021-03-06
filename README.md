# Shrine::Plugins::Transloadit

Provides [Transloadit] integration for [Shrine].

Transloadit offers advanced file processing for all sorts of media, including
images, videos, audio, and documents, along with importing from and exporting
to various file storage services.

## Setup

While Transloadit is able to export processed files to [many storage services],
this plugin currently supports only Amazon S3 (just because there are no Shrine
integrations written for other services on that list yet). You can just add
shrine-transloadit to your current setup:

```rb
gem "shrine"
gem "aws-sdk-s3", "~> 1.2" # for Amazon S3
gem "shrine-transloadit"
```

```rb
require "shrine"
require "shrine/storage/s3"

s3_options = {
  bucket: "my-bucket",
  region: "my-region",
  access_key_id: "abc",
  secret_access_key: "xyz",
}

Shrine.storages = {
  cache: Shrine::Storage::S3.new(prefix: "cache", **s3_options),
  store: Shrine::Storage::S3.new(prefix: "store", **s3_options),
}

class TransloaditUploader < Shrine
  plugin :transloadit,
    auth_key: "your transloadit key",
    auth_secret: "your transloadit secret"
end
```

This setup assumes you're doing direct S3 uploads, but you can also do [direct
uploads to Transloadit], or just use any other `:cache` storage which provides
URLs for uploaded files.

## How it works

Transloadit works in a way that you create an "assembly", which contains all
information about how the file(s) should be processed, from import to export.
Processing itself happens asynchronously, and you can give Transloadit a URL
which it will POST results to when processing finishes.

This plugin allows you to easily implement this webhook flow. You can intercept
promoting, and submit a Transloadit assembly using the cached file, along with
a URL to the route in your app where you'd like Transloadit to POST the results
of processing. Then you can call the plugin again in the route to save the
results to your attachment column.

The **[demo app]** shows a complete implementation of this flow, and can serve
as a good baseline for your own implementation.

## Usage

We loaded the `transloadit` plugin in a `TransloaditUploader` base uploader
class, so that any uploaders that you want to do Transloadit processing with
can just inherit from that class.

Transloadit assemblies are built inside `#transloadit_process` method in your
uploader, and you can use some convenient helper methods which the plugin
provides.

```rb
class ImageUploader < TransloaditUploader # inherit from our Transloadit base uploader class
  def transloadit_process(io, context)
    resized = transloadit_file(io)
      .add_step("resize", "/image/resize", width: 800)

    transloadit_assembly(resized, context: context)
  end
end
```

These helper methods just provide a higher-level interface over the
[transloadit gem], which you might want look at to get a better understanding
of how building assemblies works.

In short, in Transloadit every action, be it import, processing, or export, is
a "step". Each step is defined by its [robot and arguments], and needs to have
a *unique name*. Transloadit allows you to define the entire processing flow
(which can result in multiple files) as a collection of steps, which is called
an "assembly". Once the assembly is built it can be submitted to Transloadit.

### Versions

With Transloadit you can create multiple files in a single assembly, and this
plugin allows you to leverage that in form of a hash of versions.

```rb
class ImageUploader < TransloaditUploader
  plugin :versions

  def transloadit_process(io, context)
    original = transloadit_file(io)
    medium = original.add_step("resize_500", "/image/resize", width: 500)
    small = original.add_step("resize_300", "/image/resize", width: 300)

    files = {original: original, medium: medium, small: small}

    transloadit_assembly(files, context: context)
  end
end
```

### Webhooks

Transloadit performs its processing asynchronously, and you can provide a URL
where you want Transloadit to POST results of processing once it's finished.

```rb
class ImageUploader < TransloaditUploader
  def transloadit_process(io, context)
    # ...
    transloadit_assembly(files, notify_url: "http://myapp.com/webhooks/transloadit")
  end
end
```

Then in your `POST /webhooks/transloadit` route you can call the plugin to
automatically save the results to the attachment column in Shrine's format.

```rb
post "/webhooks/transloadit" do
  TransloaditUploader::Attacher.transloadit_save(params)
  # return 200 status
end
```

Note that if you have CSRF protection, make sure that you skip verifying the
CSRF token for this route.

### Direct uploads

Transloadit supports direct uploads, allowing you to do additional processing
on upload, along with a [jQuery plugin] for easy integration. Generally you
only want to do some light processing on direct uploads, and without any
exporting, so that you have better control over your Transloadit bandwidth.

```js
// Using https://github.com/transloadit/jquery-sdk
$("#upload-form").transloadit({
  wait: true,
  params: {
    auth: {key: "YOUR_TRANSLOADIT_AUTH_KEY"},
    steps: {
      extract_thumbnails: {
        robot:  "/video/thumbs",
        use:    ":original",
        count:  8,
        format: "png",
      }
    }
  }
});
```

When direct upload finishes, Transloadit returns information about the uploaded
file(s), one of which is a temporary URL to the file. You want to save this URL
as cached attachment, so that you can display it to the user and use it for
further Transloadit processing. You can do that using [shrine-url]:

```rb
gem "shrine-url"
```

```rb
require "shrine/storage/url"
Shrine.storages[:cache] = Shrine::Storage::Url.new
```

Now when you obtain results from finished direct uploads on the client-side,
you need to transform the Transloadit hash into Shrine's uploaded file
representation, using the URL as the "id":

```js
{
  id: data['url'], // we save the URL
  storage: 'cache',
  metadata: {
    size: data['size'],
    filename: data['name'],
    mime_type: data['mime'],
    width: data['meta'] && data['meta']['width'],
    height: data['meta'] && data['meta']['height'],
    transloadit: data['meta'],
  }
}
```

If the result of direct upload processing are multiple files (e.g. video
thumbnails), you need to assign them to individual records.

See the **[demo app]** for a complete implementation of direct uploads.

### Templates

Transloadit recommends using [templates], since they allow you to replay failed
assemblies, and also allow you not to expose credentials in your HTML.

Here is an example where the whole processing is defined inside a template,
and we just set the location of the imported file.

```rb
# Your Transloadit template saved as "my_template"
{
  steps: {
    resize: {
      robot: "/image/resize",
      use: "import", # the "import" step will be passed in
      width: 800
    },
    export: {
      robot: "/s3/store",
      use: "resize",
      bucket: "YOUR_AWS_BUCKET",
      key: "YOUR_AWS_KEY",
      secret: "YOUR_AWS_SECRET",
      bucket_region: "YOUR_AWS_REGION",
      path: "videos/${unique_prefix}/${file.url_name}"
    }
  }
}
```
```rb
class ImageUplaoder < TransloaditUploader
  def transloadit_process(io, context)
    import = transloadit_import_step("import", io)
    transloadit_assembly("my_template", steps: [import])
  end
end
```

### Backgrounding

Even though submitting a Transloadit assembly doesn't require any uploading, it
still does two HTTP requests, so you might want to put them into a background
job. You can configure that in the `TransloaditUploader` base uploader class:

```rb
class TransloaditUploader < Shrine
  plugin :transloadit,
    auth_key: "your transloadit key",
    auth_secret: "your transloadit secret"

  Attacher.promote { |data| TransloaditJob.perform_async(data) }
end
```
```rb
class TransloaditJob
  include Sidekiq::Worker

  def perform(data)
    TransloaditUploader::Attacher.transloadit_process(data)
  end
end
```

### Tracking progress

When an assembly is submitted, Transloadit returns a lot of useful information
about the status of that assembly, which the plugin saves to the cached
attachment's metadata.

```rb
response = photo.image.transloadit_response
response.body #=>
# {
#   "ok"                 => "ASSEMBLY_EXECUTING",
#   "message"            => "The assembly is currently being executed.",
#   "assembly_id"        => "83d07d10414011e68cc8c5df79919836",
#   "assembly_url"       => "http://api2.janani.transloadit.com/assemblies/83d07d10414011e68cc8c5df79919836",
#   "execution_start"    => "2016/07/03 17:06:42 GMT",
#   "execution_duration" => 2.113,
#   "params"             => "{\"steps\":{...}}",
#   ...
# }
```

At an point during the execution of the assembly you can refresh this
information:

```rb
response.finished? #=> false
response.reload!
response.finished? #=> true
```

### Metadata

For each processed file Transloadit also extracts a great deal of useful
metadata. When the Transloadit processing is finished and the results are saved
as a Shrine attachment, this metadata will be automatically used to populate
the attachment's metadata.

Additionally the Transloadit's metadata hash will be saved in an additional
metadata key, so that you can access any other values:

```rb
photo = Photo.create(image: image_file)
photo.image.metadata["transloadit"] #=>
# {
#   "date_recorded"         => "2013/09/04 08:03:39",
#   "date_file_created"     => "2013/09/04 12:03:39 GMT",
#   "date_file_modified"    => "2016/07/11 02:27:11 GMT",
#   "aspect_ratio"          => "1.504",
#   "city"                  => "Decatur",
#   "state"                 => "Georgia",
#   "country"               => "United States",
#   "latitude"              => 33.77519301,
#   "longitude"             => -84.295608,
#   "orientation"           => "Horizontal (normal)",
#   "colorspace"            => "RGB",
#   "average_color"         => "#8b8688",
#   ...
# }
```

### Import & Export

Every `TransloaditFile` needs to have an import and an export step. This plugin
automatically generates those steps for you:

```rb
transloadit_file(io)

# is equivalent to

file = transloadit_file
file.add_step(transloadit_import_step("import", io))
```

```rb
transloadit_assembly({original: original, thumb: thumb})

# is equivalent to

transloadit_assembly({
  original: original.add_step(transloadit_export_step("export_original")),
  thumb: thumb.add_step(transloadit_export_step("export_thumb")),
})
```

If you want/need to generate these steps yourself, you can just use the
expanded forms.

### Transloadit gem

If you want to have complete control over how steps are generated, you can just
use the [transloadit gem] directly. This plugin doesn't care how you generate
your steps, it only requires you to return an instance of
`Transloadit::Assembly`.

```rb
class MyUploader < TransloaditUploader
  def transloadit_process(io, context)
    # ...
    transloadit #=> #<Transloadit>
    transloadit.assembly(options)
  end
end
```

The import/export helper methods simply generate a `Transloadit::Step` object,
and you can pass additional options:

```rb
class MyUploader < TransloaditUploader
  def transloadit_process(io, context)
    transloadit_import_step("import", io)             #=> #<Transloadit::Step>
    transloadit_export_step("export", path: "mypath") #=> #<Transloadit::Step>
  end
end
```

The `#add_step` method for `TransloaditFile` is just a convenient way to add
steps where `:use` is automatically set to previous step.

### Testing

In development or test environment you cannot use webhooks, because Transloadit
as an external service cannot access your localhost. In this case you can just
do polling:

```rb
class MyUploader < TransloaditUploader
  def transloadit_process(io, context)
    # ...

    if ENV["RACK_ENV"] == "production"
      notify_url = "https://myapp.com/webhooks/transloadit"
    else
      # In development we cannot receive webhooks, because Transloadit as an
      # external service cannot reach our localhost.
    end

    transloadit_assembly(files, context: context, notify_url: notify_url)
  end
end
```

```rb
class TransloaditJob
  include Sidekiq::Worker

  def perform(data)
    attacher = TransloaditUploader::Attacher.transloadit_process(data)

    # Webhooks won't work in development, so we can just use polling.
    unless ENV["RACK_ENV"] == "production"
      response = attacher.get.transloadit_response
      response.reload_until_finished!
      attacher.transloadit_save(response.body)
    end
  end
end
```

## Contributing

Before you can run tests, you need to first create an `.env` file in the
project root containing your Transloadit and Amazon S3 credentials:

```sh
# .env
TRANSLOADIT_AUTH_KEY="..."
TRANSLOADIT_AUTH_SECRET="..."
S3_BUCKET="..."
S3_REGION="..."
S3_ACCESS_KEY_ID="..."
S3_SECRET_ACCESS_KEY="..."
```

Afterwards you can run the tests:

```sh
$ bundle exec rake test
```

## License

[MIT](/LICENSE.txt)

[Shrine]: https://github.com/janko-m/shrine
[Transloadit]: https://transloadit.com/
[many storage services]: https://transloadit.com/docs/conversion-robots/#file-export-robots
[transloadit gem]: https://github.com/transloadit/ruby-sdk
[robot and arguments]: https://transloadit.com/docs/conversion-robots/
[templates]: https://transloadit.com/docs/#templates
[jQuery plugin]: https://github.com/transloadit/jquery-sdk
[demo app]: /demo
[direct uploads to Transloadit]: #direct-uploads
[shrine-url]: https://github.com/janko-m/shrine-url
