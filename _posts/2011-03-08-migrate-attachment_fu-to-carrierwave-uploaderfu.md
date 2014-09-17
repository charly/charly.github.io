---
layout: post
title: 'Migrate Attachment_fu to CarrierWave : UploaderFu'
---

Carrierwave is my favorite gem for handling uploads and image thumbnail generation (attachment_fu died of old age and paperclip suffers from hashitis, not a pretty picture)

Anyway our imagebank was still on attachment_fu even though I upgraded to rails3 months ago. We couldn't add images and going through some dirty hacks to resuscitate attachment_fu was too much black magic.

So <a href="https://github.com/jnicklas/carrierwave">CarrierWave</a> it was.

But how to stick to attachment_fu's naming convention. CarrierWave has a nice and clean way to migrate from paperclip but none from attachment_fu, because J. Nicklas claims, they're too far a part.

Well here's a little module you can include in your uploader and adapt to your needs.

``` ruby
# Helps you migrate from attachment_fu
# put it in your /lib dir and include it your xxxx_uploader.rb
module UploaderFu

  def partition_dir
    ("%08d" % model.id).scan(/\d{4}/).join("/")
  end

  def model_dir
    "#{model.class.to_s.underscore}/#{mounted_as}/"
  end

  def root_dir
    version_name ? "public/images" : "uploads"
  end

  # store dir is composed of root_dir, model_dir, partition_dir
  # override/change any of those to fit your needs
  def store_dir
    File.join Rails.root, root_dir, model_dir, partition_dir
  end

  ### For AttachmentFu Legacy :
  # version :thumb do
  #   process :resize_to_fill => [80, 80]
  #
  #   def full_filename(for_file)
  #     full_filename_fu( super )
  #   end
  #
  # end
  def full_filename_fu( filename )
    version_prefix = "#{version_name}_"
    filename = filename.gsub(version_prefix, "")
    ext = nil
    basename = filename.gsub /\.\w+$/ do |s|
      ext = s; ''
    end
    "#{basename}_#{version_name}#{ext}"
  end

  def extension_white_list
    %w(jpg jpeg gif png)
  end

  def default_url
    File.join "/images", model_dir, default_url_filename
  end

  def default_url_filename
    [version_name, "default.jpg"].compact.join('_')
  end
end
```


Any Questions ?
