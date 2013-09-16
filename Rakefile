require 'rake/clean'
require 'erb'
require 'open-uri'

CLEAN.include("build")

DATA_VERSION="2.0.2"
DATA_URI = URI.parse("http://c758482.r82.cf2.rackcdn.com/Sublime%20Text%202.0.2%20x64.tar.bz2")
DATA_FILE = File.join("#{ENV['HOME']}/Downloads", URI.unescape(DATA_URI.path))
PACKAGE_VERSION="#{DATA_VERSION}-1"

ERB_CONFIG={
  "VERSION" => PACKAGE_VERSION
}

ERB_FILES = FileList['debian/**/*.erb'].exclude {|f| File.directory? f}
COPY_FILES = FileList['debian/**/*'].exclude {|f| File.directory? f}-ERB_FILES

ALL_FILES = ERB_FILES + COPY_FILES

file "Rakefile" do
  touch "Rakefile"
end

task "package" => "generate" do
  system %[fakeroot dpkg-deb --build build/debian build]
end

file DATA_FILE do
  download DATA_URI, DATA_FILE, :verbose => true
end

directory "build/debian/opt/lib"

file "build/.unpacked" => [DATA_FILE, "build/debian/opt/lib"] do
  puts "Unpacking #{DATA_FILE}"
  %x[tar xfj '#{DATA_FILE}' -C build/debian/opt/lib]
  touch "build/.unpacked"
end

task "generate" => "build/.unpacked"

## Create tasks for ERB files
ERB_FILES.each { |f|
  # overlay/some_file.txt.erb -> overlays/production/some_file.txt
  target = target = f.pathmap("build/%X")
  target_dir = File.dirname target
  directory target_dir
  file target => [target_dir, f, "Rakefile"] do
    erb_template f, target, ERB_CONFIG
  end
  task "generate" => target 
}

## Create tasks to copy other files
COPY_FILES.each { |f|
  # overlay/some_file.txt -> overlays/production/some_file.txt
      target = f.pathmap("build/%p")
  target_dir = File.dirname target
  directory target_dir
  if File.symlink?(f)
    file f do
    end
  end
  file target => [target_dir, f] do
    system %[cp -vPp '#{f}' '#{target}']
  end
  task "generate" => target
}

def erb_template(template_file, dest_file, params)
  puts "#{template_file} -> #{dest_file}"
  template = ERB.new File.new(template_file).read, nil, '-'
  File.open(dest_file, 'w') {|f| f.write template.result(binding) }
end



def download(src, dst, options = {})
  options = {
    :verbose => false,
    :force => false,
    :create_parents => false
    }.merge(options)

  src = URI.parse(src) if !src.is_a? URI
  pbar = nil

  if !File.exists?(dst) && options[:create_parents]
    mkdir_p File.basename(dst)
  end

  if File.directory? dst
    dst = File.join dst, File.basename(src.path)
  end

  if File.exists?(dst) && !options[:force]
    puts "#{dst}: already downloaded." if options[:verbose]
    return
  end

  pbar = nil
  open(dst, 'wb') do |fo|
    begin
      fo.print open(src,
        :content_length_proc => lambda { |t|
          pbar = ProgressBar.new(dst, t) if options[:verbose]
        },

        :progress_proc => lambda {|s|
            pbar.set s if pbar
        }).read
    rescue Exception => e
      rm dst, :verbose => false
      raise "Cannot download: #{src}: #{e.message}"
    end
  end
  puts
end


class ProgressBar
  def initialize(title, max)
    @title = title
    @max = max
    @current = 0
    @spinner = ['-', '\\', '|', '/']
  end

  def set(current)
    if @max
      @current = current
    else
      @current = @current + 1
    end
    print    
  end

  def print
    if @max
      STDOUT.print "#{@title}: #{(@current*100)/@max}/100%\r"
    else
      STDOUT.print "#{@title}: #{@spinner[@current % @spinner.size]}\r"
    end
  end
end
