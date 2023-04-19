# CollectionBuilder-CSV helper tasks

require 'csv'
require 'image_optim'
require 'mini_magick'
require 'fileutils'

###############################################################################
# TASK: deploy
###############################################################################

desc "Build site with production env"
task :deploy do
  ENV["JEKYLL_ENV"] = "production"
  exec("bundle exec jekyll build")
end

###############################################################################
# Helper Functions
###############################################################################

def prompt_user_for_confirmation(message)
  response = nil
  loop do
    print "#{message} (Y/n): "
    $stdout.flush
    response = case STDIN.gets.chomp.downcase
               when "", "y" then true
               when "n" then false
               end
    break if response != nil
    puts "Please enter \"y\" or \"n\""
  end
  response
end

def process_and_optimize_image(filename, file_type, output_filename, size, density)
  if filename == output_filename
    puts "Optimizing: #{filename}"
    image_optim = ImageOptim.new(:svgo => false)
    image_optim.optimize_image!(filename)
  else
    puts "Creating: #{output_filename}"
    begin
      image = MiniMagick::Image.open(filename)
      image.format("jpg") if file_type == :pdf
      image.combine_options do |i|
        i.density(density) if file_type == :pdf
        i.resize(size)
        i.flatten
      end
      image.write(output_filename)
      image_optim = ImageOptim.new(:svgo => false)
      image_optim.optimize_image!(output_filename)
      # image.data = image_optim.optimize_image_data(image.data)
      # image.write(output_filename)
    rescue => e
      puts "Error creating #{filename}: #{e.message}"
    end
  end
end

###############################################################################
# TASK: generate_derivatives
###############################################################################

desc "Generate derivative image files from collection objects"
task :generate_derivatives, [:thumbs_size, :small_size, :density, :missing, :compress_originals] do |t, args|

  # set default arguments 
  args.with_defaults(
    :thumbs_size => "300x300",
    :small_size => "800x800",
    :density => "300",
    :missing => "true",
    :compress_originals => "false",
  )

  # set the folder locations
  objects_dir = "objects"
  thumb_image_dir = "objects/thumbs"
  small_image_dir = "objects/small"

  # Ensure that the output directories exist.
  [thumb_image_dir, small_image_dir].each do |dir|
    Dir.mkdir(dir) unless Dir.exists?(dir)
  end

  # support these file types
  EXTNAME_TYPE_MAP = {
    '.tiff' => :image,
    '.tif' => :image,  
    '.jpg' => :image,
    '.png' => :image,
    '.pdf' => :pdf
  }

  # CSV output
  list_name = File.join(objects_dir, "object_list.csv")
  field_names = "object_location,image_small,image_thumb".split(",")
  CSV.open(list_name, "w") do |csv|
    csv << field_names

    # Iterate over all files in the objects directory.
    Dir.glob(File.join(objects_dir, '*')).each do |filename|
      # Skip subdirectories and the README.md file.
      next if File.directory?(filename) || File.basename(filename) == "README.md" || File.basename(filename) == "object_list.csv"

      # Determine the file type and skip if unsupported.
      extname = File.extname(filename).downcase
      file_type = EXTNAME_TYPE_MAP[extname]
      if !file_type
        puts "Skipping file with unsupported extension: #{filename}"
        csv << ["/" + filename, nil, nil]
        next
      end

      # Get the lowercase filename without any leading path and extension.
      base_filename = File.basename(filename, ".*").downcase

      # Optimize the original image.
      if args.compress_originals == 'true'
        puts "Optimizing: #{filename}"
        image_optim.optimize_image!(filename)
      end

      # Generate the thumb image.
      thumb_filename = File.join(thumb_image_dir, "#{base_filename}_th.jpg")
      if args.missing == 'false' || !File.exists?(thumb_filename)
        process_and_optimize_image(filename, file_type, thumb_filename, args.thumbs_size, args.density)
      else
        puts "Skipping: #{thumb_filename} already exists"
      end

      # Generate the small image.
      small_filename = File.join([small_image_dir, "#{base_filename}_sm.jpg"])
      if args.missing == 'false' or !File.exists?(small_filename)
        process_and_optimize_image(filename, file_type, small_filename, args.small_size, args.density)
      else
        puts "Skipping: #{small_filename} already exists"
      end
      csv << ["/"+filename,"/"+small_filename,"/"+thumb_filename]
    end
  end
  puts "\e[32mSee '#{list_name}' for list of objects and derivatives created.\e[0m"
end
