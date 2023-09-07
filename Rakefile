# frozen_string_literal: true

# CollectionBuilder related helper tasks

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
  args.with_defaults(
    thumbs_size: '300x300',
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
# TASK: rename_by_csv
#
# read csv, rename files
#
###############################################################################

desc "rename objects using csv"
task :rename_by_csv, [:csv_file,:filename_current,:filename_new,:input_dir,:output_dir] do |_t, args|
  # set default arguments
  args.with_defaults(
    csv_file: 'rename.csv',
    filename_current: 'filename_old',
    filename_new: 'filename_new',
    input_dir: 'objects/',
    output_dir: 'renamed/'
  )

  # check for csv file
  if !File.exist?(args.csv_file)
    puts "CSV file does not exist! No files renamed and exiting."
  else
    # read csv file
    csv_text = File.read(args.csv_file, :encoding => 'utf-8')
    csv_contents = CSV.parse(csv_text, headers: true)

    # Ensure that the output directory exists.
    FileUtils.mkdir_p(args.output_dir) unless Dir.exist?(args.output_dir)

    # iterate on csv rows
    csv_contents.each do |item|
      # check csv for old and new filenames
      if item[args.filename_current]
        name_old = File.join(args.input_dir, item[args.filename_current])
      else
        puts "no current filename given, skipping!"
        next
      end
      if item[args.filename_new]
        name_new = File.join(args.output_dir, item[args.filename_new])
      else
        puts "no new filename given, skipping!"
        next
      end
      # check if old and new file exist
      if !File.exist?(name_old)
        puts "old file '#{name_old}' does not exist, skipping!"
        next
      end
      if File.exist?(name_new)
        puts "new filename '#{name_new}' already exists, skipping!"
        next
      end
      puts "copying: '#{name_old}' to '#{name_new}'"
      system('cp', name_old, name_new)
    end
    
    puts "done renaming."

  end

end




###############################################################################
# TASK: move_by_csv
#
# read csv, move files
#
###############################################################################

desc "move objects from csv list"
task :move_by_csv, [:csv_file,:filename_current,:input_dir,:output_dir] do |_t, args|
  # set default arguments
  args.with_defaults(
    csv_file: 'move.csv',
    filename_current: 'filename',
    input_dir: 'objects/',
    output_dir: 'moved/'
  )

  # check for csv file
  if !File.exist?(args.csv_file)
    puts "CSV file does not exist! No files renamed and exiting."
  else
    # read csv file
    csv_text = File.read(args.csv_file, :encoding => 'utf-8')
    csv_contents = CSV.parse(csv_text, headers: true)

    # Ensure that the output directory exists.
    FileUtils.mkdir_p(args.output_dir) unless Dir.exist?(args.output_dir)

    # iterate on csv rows
    csv_contents.each do |item|
      # check for filename
      if item[args.filename_current]
        name_old = File.join(args.input_dir, item[args.filename_current])
      else
        puts "no current filename given, skipping!"
        next
      end
      # set new file name
      name_new = File.join(args.output_dir, item[args.filename_current])
      # check if old and new file exist
      if !File.exist?(name_old)
        puts "old file '#{name_old}' does not exist, skipping!"
        next
      end
      if File.exist?(name_new)
        puts "new filename '#{name_new}' already exists, skipping!"
        next
      end
      # move the file by copying
      puts "copying: '#{name_old}' to '#{name_new}'"
      system('cp', name_old, name_new)
    end
    
    puts "done moving."

  end

end




###############################################################################
# TASK: rename_lowercase
#
# copy files, rename all files to lowercase
#
###############################################################################

desc "rename lowercase"
task :rename_lowercase, [:input_dir,:output_dir] do |_t, args|
  # set default arguments
  args.with_defaults(
    input_dir: 'objects/',
    output_dir: 'renamed/'
  )

  # ensure input directory exists
  if !Dir.exist?(args.input_dir)
    puts "Input folder does not exist!"
    break
  end

  # ensure that the output directory exists.
  FileUtils.mkdir_p(args.output_dir) unless Dir.exist?(args.output_dir)
  
  # Generate derivatives.
  Dir.glob(File.join([args.input_dir, '*'])).each do |filename|
    # Ignore subdirectories.
    if File.directory? filename
      next
    end

    # Get the lowercase filename 
    name_old = filename
    name_new = File.join(args.output_dir, File.basename(filename).downcase)

    # check if file already exists
    if File.exist?(name_new)
      puts "new filename '#{name_new}' already exists, skipping!"
      next
    end

    # copy file
    puts "renaming: '#{name_old}' to '#{name_new}'"
    system('cp', name_old, name_new)

  end
end

###############################################################################
# TASK: download_by_csv
#
# read csv, download using wget
#
###############################################################################

desc "download objects and rename using csv"
task :download_by_csv, [:csv_file,:download_link,:download_rename,:output_dir] do |_t, args|
  # set default arguments
  args.with_defaults(
    csv_file: 'download.csv',
    download_link: 'url',
    download_rename: 'filename_new',
    output_dir: 'download/'
  )

  # check for csv file
  if !File.exist?(args.csv_file)
    puts "CSV file does not exist! No files downloaded and exiting."
  else
    # read csv file
    csv_text = File.read(args.csv_file, :encoding => 'utf-8')
    csv_contents = CSV.parse(csv_text, headers: true)

    # Ensure that the output directory exists.
    FileUtils.mkdir_p(args.output_dir) unless Dir.exist?(args.output_dir)    
        
    # iterate on csv rows
    csv_contents.each do |item|
      # check for download url
      if item[args.download_link]
        # check for rename
        if item[args.download_rename]
          # check if file already exists
          name_new = File.join(args.output_dir, item[args.download_rename])
          if File.exist?(name_new)
            puts "new filename '#{name_new}' already exists, skipping!"
            next
          end
          puts "downloading"
          # call wget
          system('wget','-O', name_new, item[args.download_link])
        else
          puts "downloading"
          # call wget 
          system('wget', item[args.download_link], "-P", args.output_dir)
        end
      else
        puts "no download url!"
      end
    end

    puts "done downloading."

  end

end
