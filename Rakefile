# frozen_string_literal: true

# CollectionBuilder-CSV helper tasks

require 'csv'
require 'fileutils'
require 'image_optim' unless Gem.win_platform?
require 'mini_magick'

###############################################################################
# TASK: deploy
###############################################################################

desc 'Build site with production env'
task :deploy do
  ENV['JEKYLL_ENV'] = 'production'
  system('bundle', 'exec', 'jekyll', 'build')
end

###############################################################################
# Helper Functions
###############################################################################

def prompt_user_for_confirmation(message)
  response = nil
  loop do
    print "#{message} (Y/n): "
    $stdout.flush
    response = case $stdin.gets.chomp.downcase
               when '', 'y' then true
               when 'n' then false
               end
    break unless response.nil?

    puts 'Please enter "y" or "n"'
  end
  response
end

def process_and_optimize_image(filename, file_type, output_filename, size, density)
  image_optim = ImageOptim.new(svgo: false) unless Gem.win_platform?
  if filename == output_filename && file_type == :image && !Gem.win_platform?
    puts "Optimizing: #{filename}"
    begin
      image_optim.optimize_image!(output_filename)
    rescue StandardError => e
      puts "Error optimizing #{filename}: #{e.message}"
    end
  elsif filename == output_filename && file_type == :pdf
    puts "Skipping: #{filename}"
  else
    puts "Creating: #{output_filename}"
    begin
      if file_type == :pdf
        inputfile = "#{filename}[0]"
        magick = MiniMagick::Tool::Convert.new
        magick.density(density)
        magick << inputfile
        magick.resize(size)
        magick.flatten
        magick << output_filename
        magick.call
      else
        image = MiniMagick::Image.open(filename)
        image.format('jpg')
        image.resize(size)
        image.flatten
        image.write(output_filename)
      end
      image_optim.optimize_image!(output_filename) unless Gem.win_platform?
    rescue StandardError => e
      puts "Error creating #{filename}: #{e.message}"
    end
  end
end

###############################################################################
# TASK: generate_derivatives
###############################################################################

desc 'Generate derivative image files from collection objects'
task :generate_derivatives, [:thumbs_size, :small_size, :density, :missing, :compress_originals] do |_t, args|
  # set default arguments
  # default image size is based on max pixel width they will appear in the base template features
  args.with_defaults(
    thumbs_size: '450x',
    small_size: '800x800',
    density: '300',
    missing: 'true',
    compress_originals: 'false'
  )

  # set the folder locations
  objects_dir = 'objects'
  thumb_image_dir = 'objects/thumbs'
  small_image_dir = 'objects/small'

  # Ensure that the output directories exist.
  [thumb_image_dir, small_image_dir].each do |dir|
    FileUtils.mkdir_p(dir) unless Dir.exist?(dir)
  end

  # support these file types
  EXTNAME_TYPE_MAP = {
    '.jpeg' => :image,
    '.jpg' => :image,
    '.pdf' => :pdf,
    '.png' => :image,
    '.tif' => :image,
    '.tiff' => :image
  }.freeze

  # CSV output
  list_name = File.join(objects_dir, 'object_list.csv')
  field_names = 'filename,object_location,image_small,image_thumb'.split(',')
  CSV.open(list_name, 'w') do |csv|
    csv << field_names

    # Iterate over all files in the objects directory.
    Dir.glob(File.join(objects_dir, '*')).each do |filename|
      # Skip subdirectories and the README.md file.
      if File.directory?(filename) || File.basename(filename) == 'README.md' || File.basename(filename) == 'object_list.csv'
        next
      end

      # Determine the file type and skip if unsupported.
      extname = File.extname(filename).downcase
      file_type = EXTNAME_TYPE_MAP[extname]
      unless file_type
        puts "Skipping file with unsupported extension: #{filename}"
        csv << ["#{File.basename(filename)}", "/#{filename}", nil, nil]
        next
      end

      # Get the lowercase filename without any leading path and extension.
      base_filename = File.basename(filename, '.*').downcase

      # Optimize the original image.
      if args.compress_originals == 'true'
        puts "Optimizing: #{filename}"
        process_and_optimize_image(filename, file_type, filename, nil, nil)
      end

      # Generate the thumb image.
      thumb_filename = File.join(thumb_image_dir, "#{base_filename}_th.jpg")
      if args.missing == 'false' || !File.exist?(thumb_filename)
        process_and_optimize_image(filename, file_type, thumb_filename, args.thumbs_size, args.density)
      else
        puts "Skipping: #{thumb_filename} already exists"
      end

      # Generate the small image.
      small_filename = File.join([small_image_dir, "#{base_filename}_sm.jpg"])
      if (args.missing == 'false') || !File.exist?(small_filename)
        process_and_optimize_image(filename, file_type, small_filename, args.small_size, args.density)
      else
        puts "Skipping: #{small_filename} already exists"
      end
      csv << ["#{File.basename(filename)}", "/#{filename}", "/#{small_filename}", "/#{thumb_filename}"]
    end
  end
  puts "\e[32mSee '#{list_name}' for list of objects and derivatives created.\e[0m"
end

###############################################################################
# TASK: generate_manifests
###############################################################################

desc "Generate IIIF manifest files from collection objects"
task :generate_manifests do

  # Load _config.yml file
  config = YAML.load_file('_config.yml')

  # Define the base URL
  base_url = config['baseurl'] || ''

  # Get metadata csv value
  csv_file_path = "_data/#{config['metadata']}.csv"
  raise "File #{csv_file_path} does not exist" unless File.exist?(csv_file_path)

  # Create 'iiif' directory within the 'objects' directory
  iiif_directory = File.join('objects', 'iiif')
  FileUtils.mkdir_p(iiif_directory) unless File.directory?(iiif_directory)

  # Initialize a hash to store parent-child relationships and standalone objects
  objects = {}
  CSV.foreach(csv_file_path, headers: true) do |row|
    if row['parentid'].to_s.strip.empty?
      # It's either a standalone object or a parent
      objects[row['objectid']] ||= {row: row, children: []}
    else
      # It's a child object
      objects[row['parentid']] ||= {row: nil, children: []}
      objects[row['parentid']][:children] << row
    end
  end
  
  # Generate manifests for parents and standalone objects only
  objects.each do |objectid, data|
    next if data[:row].nil?  # Skip if the object is a child

    parent_row = data[:row]
    children = data[:children]

    # Define manifest structure
    manifest = {
      "@context": "http://iiif.io/api/presentation/3/context.json",
      "id": "#{base_url}/iiif/#{parent_row['objectid']}/manifest.json",
      "type": "Manifest",
      "label": {
        "en": [parent_row['title']]
      },
      # "attribution": parent_row['contributing_institution'],
      # "logo": "#{base_url}/assets/img/collectionbuilder-logo.png",
      "thumbnail": [
        {
          "id": "#{base_url}#{parent_row['image_thumb']}",
          "type": "Image",
          "format": "image/jpeg",
          "height": 681,
          "width": 450
        }
      ],
      "items": []
    }

    # Add children as items if any, otherwise add self
    objects = children.any? ? children : [parent_row]
    objects.each do |row|
      next unless row['object_location'] # Skip if no object_location
      items = {
        "id": "https://example.org/6124c883-6869-4ec3-8286-9f3c6d2f0327/canvas/#{row['objectid']}",
        "type": "Canvas",
        "height": 1513,
        "width": 1000,
        "label": {
          "en": [
            row['title']
          ]
        },
        "thumbnail": [
          {
            "id": "#{base_url}#{row['image_thumb']}",
            "type": "Image",
            "format": "image/jpeg",
            "height": 681,
            "width": 450
          }
        ],
        "items": [
          {
            "id": "https://example.org/6124c883-6869-4ec3-8286-9f3c6d2f0327/canvas/#{row['objectid']}",
            "type": "AnnotationPage",
            "items": [
              {
                "id": "https://example.org/6124c883-6869-4ec3-8286-9f3c6d2f0327/annotation/#{row['objectid']}",
                "type": "Annotation",
                "motivation": "painting",
                "target": "https://example.org/6124c883-6869-4ec3-8286-9f3c6d2f0327/canvas/#{row['objectid']}",
                "body": {
                  "id":  "#{base_url}#{row['object_location']}",
                  "type": "Image",
                  "format": "image/jpeg",
                  "height": 1513,
                  "width": 1000
                }
              }
            ]
          }
        ]
      }
      # Add rendering section if object_ocr is present
      # if row['object_ocr'] && !row['object_ocr'].strip.empty?
        # rendering_section = {
          # "rendering": [
            # {
              # "@id": row['object_ocr'],
              # "format": "text/plain",
              # "label": "Raw OCR Data"
            # }
          # ]
        # }
        # items.merge!(rendering_section)
      # end
      # Add items to the sequence array
      manifest[:items] << items
    end

    # Save manifest in the object's subdirectory
    subdirectory_path = File.join(iiif_directory, parent_row['objectid'])
    FileUtils.mkdir_p(subdirectory_path) unless File.directory?(subdirectory_path)
    file_path = File.join(subdirectory_path, "manifest.json")
    File.open(file_path, "w") do |file|
      file.write(JSON.pretty_generate(manifest))
    end
  end

  puts "IIIF manifest files created for parents and standalone objects"
end