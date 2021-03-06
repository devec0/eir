# Rake-based build system for Eir
# Downloads, verifies, and builds sources into a usable toolkit, which is then used to compile a minimal distro.
#
# Author:: James Hebden (james@ec0.io)
# Copyright:: Copyleft 2016 James Hebden
# License:: GPLv3

require 'erb'
require 'yaml'
require 'open-uri'
require 'progressbar'
require 'digest'
require 'zlib'
require 'xz'
require 'bzip2/ffi'
require 'colorize'
#require 'archive/tar/minitar'
require 'rubygems/package'
require 'facter'

#Suppress XZ deprecation notices
XZ.disable_deprecation_notices = true

#Get list of packages
puts "Loading packages from packages/*.yml".green
@packages = Rake::FileList.new( "packages/*.yml" )

#Get package longname (e.g. package-x.yz)
def get_package_longname ( source )

  #get basename (just the filename part of the path)
  name = File.basename( source )

  #Remove anything after a dot followed by at least two ASCII characters.
  #This will strip any extensions we're interested in passing to this function,
  #whilst preserving version information in files.
  #We care about the version information, becuase the loose standard for 
  #tarballs is for the archive to be extracted into a folder name somethign
  #along the lines of <project name>-<version>
  name.sub(/\.[a-z][a-z].*/, '')

end

#Get package configuration from YML metadata
def get_package_data ( package )

  #Load data from YML
  package_data = YAML.load_file( package ) 

  #Do we have a prefix directory?
  if package_data[ 'prefix' ].nil?
    package_data[ 'longname' ] = ''
  else
    package_data[ 'longname' ] = "#{ package_data[ 'prefix' ] }/"
  end

  #Get package name, basename and metadata
  package_data[ 'longname' ] << get_package_longname( package_data[ 'file' ] ) 

  #Set environment variables for package
  ENV["EIR_#{ package_data[ 'name' ].upcase }_VERSION"] = package_data[ 'version' ]
  ENV["EIR_#{ package_data[ 'name' ].upcase }_SOURCE"] = "#{ File.dirname(__FILE__) }/build/#{ package_data[ 'longname' ] }"

  package_data

end

#Packages with metadata
@package_metadata = Hash.new 
@packages.each do |package|
  @package_metadata[ package ] = get_package_data( package )
end

#Download file via HTTP
def get_package_file ( dest, uri )

  #Open destination file
  File.open( dest, "w" ) do |dest_file|

    progress_bar = nil

    #Write file from open-uri initiated download file handle, register callbacks for progress bar.
    dest_file.write open( uri,
      :content_length_proc => lambda { |size|
        if size && size > 0
          progress_bar = ProgressBar.new( "...", size )
          progress_bar.file_transfer_mode
        end
      },
      :progress_proc => lambda { |position|
          progress_bar.set position if progress_bar
      }
    ).read 
  end
end

#Return hash of downloaded file, as string
def hash_package_file ( dest )

  "#{ Digest::SHA256.file( dest ) }"

end

#Open a decompression stream/file handle based on archive time, and register decompressor if appropriate.
def get_tar_handle ( source )

  type = File.extname( source )

  #switch on last part of filename to determine archive type. Faster than using libmagic, 
  #but obviously less reliable if people do weird things with archives.
  begin
    case type
    when ".tar"
      tar_handle = File.open( source )
    when ".gz"
      gz_file = File.open( source )
      tar_handle = Zlib::GzipReader.new( gz_file )
    when ".bz2"
      bz2_file = File.open ( source )
      tar_handle = StringIO.new ( Bzip2::FFI::Reader.open( bz2_file ).read )
    when ".xz"
      tar_handle = XZ::StreamReader.open( source )
    else
      raise "Unknown archive type: #{ source }"
    end
  rescue
    raise "Error extracting #{ source }"
  end

  #return file handle to decompressor or tar stream
  tar_handle

end

#Extact archive
def extract_archive ( source, dest )

  #get decompressor handle
  tar_handle = get_tar_handle( source )
  #unpack files from archive
  Gem::Package::TarReader.new( tar_handle ) do |tar|
    tar.each do |entry|
      if entry.directory?
        FileUtils.mkdir_p( "#{ dest }/#{ entry.full_name }" )
      elsif entry.file?
        FileUtils.mkdir_p( "#{ dest }/#{ File.dirname( entry.full_name ) }")
        #skip errors here, but show the file in orange
        begin
          File.open( "#{ dest }/#{ entry.full_name }", "wb" ) do |f|
            f.write( entry.read )
          end
        rescue
          puts entry.full_name.yellow
        else
          puts entry.full_name.green
        end
        File.chmod( entry.header.mode, "#{ dest }/#{ entry.full_name }")
      end
    end
  end
end

#Generate package tasks for each package.
#Tasks include:
# - A file task for downloading the source package
# - A task that verifies the file against the hash in the project metadata
# - A directory task that extracts the source if it has been downloaded and verified
# - A task that builds & installs the project, using options from each "stage" - i.e. tools build, live root build.
puts "Processing package metadata and building tasks for packages".green
@packages.each do |package|

  #Generate per-package tasks

  #Download package via HTTP
  desc "Download #{ @package_metadata[ package ][ 'file' ] }"
  file "source/#{ @package_metadata[ package][ 'file' ] }" do

    puts "Downloading #{ @package_metadata[ package][ 'file' ] } from #{ @package_metadata[ package][ 'uri' ] }".blue 
    get_package_file( "source/#{ @package_metadata[ package][ 'file' ] }", @package_metadata[ package][ 'uri' ] )

  end

  #Verify source, then extract package by getting appropriate decompressor handle, and passing to tarreader.
  #TODO: Write better tar handler to give us more control over decompressor handling, as XZ needs special handling.
  desc "Verify & extract #{ @package_metadata[ package][ 'file' ] } to build/#{ @package_metadata[ package][ 'longname' ] }"
  directory "build/#{ @package_metadata[ package][ 'longname' ] }" => "source/#{ @package_metadata[ package ][ 'file' ] }" do 

    hash = hash_package_file( "source/#{ @package_metadata[ package ][ 'file' ] }" )
    puts "Checking hash for #{ @package_metadata[ package ][ 'file' ] } (#{ hash }) against expected hash (#{ @package_metadata[ package ][ 'hash' ] })".blue
    begin
     raise "Hash failed to verify".red if @package_metadata[ package ][ 'hash' ] != hash 
    end
    puts "Hash verified OK. Going to extract.".blue

    puts "Extracting #{ @package_metadata[ package][ 'file' ] } to build/#{ @package_metadata[ package][ 'longname' ] }".green
    extract_archive( "source/#{ @package_metadata[ package][ 'file' ] }", "build/#{ @package_metadata[ package ][ 'prefix' ] }") 
    puts "Extraction finished.".blue
    
  end 

  #Build package based on options in metadata.
  #Accepts package build phase (initial, toolchain, final) as argument.
  if @package_metadata[ package ].key?( 'build' )
    @package_metadata[ package ][ 'build' ].each do |phase,command|

      desc "Patch package #{ @package_metadata[ package ][ 'name' ] } for phase #{ phase }"
      task "patch_#{ phase }_#{ @package_metadata[ package ][ 'name' ] }".to_sym => "stamps/.stamp_patch_#{ phase }_#{ @package_metadata[ package ][ 'name' ] }"

      desc "Place stamp for package #{ @package_metadata[ package ][ 'name' ] } for phase #{ phase } once built"
      file "stamps/.stamp_patch_#{ phase }_#{ @package_metadata[ package ][ 'name' ] }" do

        puts "Patching #{ @package_metadata[ package ][ 'name' ]} for phase #{ phase }".green
        puts "Checking for patches in patches/#{ phase }/#{ @package_metadata[ package ][ 'longname' ] }".light_black

        if File.directory?( "patches/#{ phase }/#{ @package_metadata[ package ][ 'longname' ] }" ) then
          Dir.glob( "patches/#{ phase }/#{ @package_metadata[ package ][ 'longname' ] }/*.patch" ) do |patch|

            patch_result = false

            puts "Applying patch #{ patch }".blue
            Dir.chdir( "build/#{ @package_metadata[ package ][ 'longname' ] }" ){
              patch_result = system("patch -N -p1 -i #{ ENV['EIR'] }/#{ patch }")
            }

            raise "Error applying patch #{ patch } to #{ @package_metadata[ package ][ 'longname' ]}".red unless patch_result 

          end
        end 

        FileUtils::touch("stamps/.stamp_patch_#{ phase }_#{ @package_metadata[ package ][ 'name' ] }")       
 
      end

      desc "Build package #{ @package_metadata[ package][ 'name' ] } for phase #{ phase }"
      task "build_#{ phase }_#{ @package_metadata[ package][ 'name' ] }".to_sym => "stamps/.stamp_build_#{ phase }_#{ @package_metadata[ package ][ 'name' ] }" 

      desc "Place stamp for package #{ @package_metadata[ package][ 'name' ] } for phase #{ phase } once built"
      file "stamps/.stamp_build_#{ phase }_#{ @package_metadata[ package ][ 'name' ] }" do

        #Fetch build command, skip if it doesn't exist for this stage.
        puts "Building #{ @package_metadata[ package ][ 'name' ]} for phase #{ phase }".green
        puts "Running #{ @package_metadata[ package ][ 'build' ][ phase ] } in build/#{ phase }".light_black

        #Set env variables
        ENV["EIR_#{ @package_metadata[ package ][ 'name' ].upcase }_BUILD"] = "#{ File.dirname(__FILE__) }/build/#{ phase }/#{ @package_metadata[ package ][ 'longname' ] }"

        #Print CFLAGS
        if ( ENV.has_key?( 'CFLAGS' ) ) then
          puts "Detected additional CFLAGS: #{ ENV['CFLAGS'] }".blue
          ENV['CXXFLAGS'] = ENV['CFLAGS']
        end

        #Set environment variables so long as we're not in initial phase
        case phase.to_s 
        when "cross", "cross32" then 
        
          puts "Setting first cross-compilation phase variables".green
          ENV['PATH'] = ENV['EIR_PATH']
          puts "Set path to #{ ENV['PATH'] }".blue

        when "initial" then
          
          puts "Setting initial toolchain variables".green

          ENV['PATH'] = ENV['EIR_PATH']
          ENV['AR'] = "#{ ENV['EIR_TARGET'] }-ar"
          ENV['AS'] = "#{ ENV['EIR_TARGET'] }-as"
          ENV['LD'] = "#{ ENV['EIR_TARGET'] }-ld"
          ENV['NM'] = "#{ ENV['EIR_TARGET'] }-nm"
          ENV['CC'] = "#{ ENV['EIR_TARGET'] }-gcc"
          ENV['CXX'] = "#{ ENV['EIR_TARGET'] }-g++"
          ENV['RANLIB'] = "#{ ENV['EIR_TARGET'] }-ranlib"
          ENV['STRIP'] = "#{ ENV['EIR_TARGET'] }-strip"
          ENV['OBJCOPY'] = "#{ ENV['EIR_TARGET'] }-objcopy"
          ENV['OBJDUMP'] = "#{ ENV['EIR_TARGET'] }-objdump"

        when "final" then

        else
      
          puts "Unknown phase: #{ phase }".light_black

        end

        #Make sure build directory exists for this stage
        FileUtils::mkdir_p( "build/#{ phase }/#{ @package_metadata[ package ][ 'longname' ] }" )

        #Set default return status
        build_result = false 

        #touch automake files if they exist
        %w[configure.ac aclocal.m4 configure Makefile.am Makefile.in].each do |file|
          if File.exist?( "build/#{ @package_metadata[ package ][ 'longname' ] }/#{ file }" ) then
            puts "Updating modification time on #{ file }".light_black
            FileUtils.touch( "build/#{ @package_metadata[ package ][ 'longname' ] }/#{ file }" )
          end
        end
        
        #Run build command in appropriate directory
        Dir.chdir( "build/#{ phase }/#{ @package_metadata[ package ][ 'longname' ] }" ){
          build_result = system("#{ @package_metadata[ package ][ 'build' ][ phase ] }")
        }

        #Check return and set stamp if needed
        if build_result then

          FileUtils.touch("stamps/.stamp_build_#{ phase }_#{ @package_metadata[ package ][ 'name' ] }")

        else

          raise "#{ @package_metadata[ package ][ 'longname' ] } failed to build".red 

        end
      end
    end
  else
      puts "No build commands available for package #{ package }"
  end

  #Generate temporary dependency on extract task, until these can be codified into stages
  #Eventual dependency chain will be build -> stage task -> build_package -> extract package -> verify source -> download source
  task "#{ package }" => [ "build/#{ @package_metadata[ package][ 'longname' ] }" ] 

end

directory "build/" do

  FileUtils.mkdir_p( "build" )

end

directory "stamps/" do

  FileUtils.mkdir_p( "stamps" )

end

directory "source/" do

  FileUtils.mkdir_p( "source" )

end

directory "output/" do

  FileUtils.mkdir_p( "output" )

end

task :clean do

  %w[
    output
    build/cross
    build/cross32
    build/initial
  ].each do |directory|
    puts "Removing #{ directory }".red
    FileUtils.remove_dir( directory, :force => true )
  end

  Dir.glob( 'stamps/.stamp_build_*' ).each do |stamp|
    puts "Clearing stamp #{ stamp }".red
    File.delete( stamp )
  end

end

task :build_root_image do
end

task :build_iso do
end

task :build_toolchain => [
  :build_cross_file,
  :build_cross_linux,
  :build_cross_m4,
  :build_cross_ncurses,
  :build_cross_gmp,
  :build_cross_mpfr,
  :build_cross_mpc,
  :build_cross_isl,
  :patch_cross_binutils,
  :build_cross_binutils,
  :patch_cross_gccstatic,
  :build_cross_gccstatic,
  :build_cross_glibc,
  :build_cross32_glibc,
  :patch_cross_gcc,
  :build_cross_gcc,
  :build_initial_binutils,
  :build_initial_gmp,
  :build_initial_mpfr,
  :build_initial_mpc,
  :build_initial_isl,
  :build_initial_zlib,
  :build_initial_gcc,
  :patch_initial_ncurses,
  :build_initial_ncurses,
  :build_initial_bash,
  :build_initial_bzip2,
  :patch_initial_coreutils,
  :build_initial_coreutils,
  :build_initial_diffutils,
  :build_initial_file,
  :build_initial_findutils,
  :build_initial_gawk,
  :build_initial_gettext,
  :build_initial_grep,
  :build_initial_gzip,
  :build_initial_make,
  :build_initial_patch,
  :patch_initial_sed,
  :build_initial_sed,
  :build_initial_tar,
  :patch_initial_texinfo,
  :build_initial_texinfo,
  :build_initial_pam,
  :build_initial_busybox,
  :build_initial_xz,
  :build_initial_ruby
] do
end

task :prepare_packages => @packages do

  puts "Checking and preparing package sources".green

end 

task :env do

  ENV['EIR'] = File.dirname(__FILE__)
  puts "Set $EIR environment variable to #{ ENV['EIR'] }".blue

  ENV['EIR_PREFIX'] = "#{ ENV['EIR'] }/output"
  puts "Set $EIR_PREFIX environment variable to #{ ENV['EIR_PREFIX'] }".blue

  ENV['EIR_TOOLCHAIN_PREFIX'] = "#{ ENV['EIR_PREFIX'] }/toolchain"
  puts "Set $EIR_TOOLCHAIN_PREFIX environment variable to #{ ENV['EIR_TOOLCHAIN_PREFIX'] }".blue

  ENV['EIR_CROSS_TOOLCHAIN_PREFIX'] = "#{ ENV['EIR_PREFIX'] }/cross-toolchain"
  puts "Set $EIR_CROSS_TOOLCHAIN_PREFIX environment variable to #{ ENV['EIR_CROSS_TOOLCHAIN_PREFIX'] }".blue
 
  ENV['EIR_PATH'] = "#{ ENV['EIR_TOOLCHAIN_PREFIX'] }/bin:#{ ENV['EIR_CROSS_TOOLCHAIN_PREFIX'] }/bin:/bin:/usr/bin"
  puts "Set $EIR_PATH environment variable to #{ ENV['EIR_PATH'] }".blue
 
  ENV['EIR_CORES'] = ( Facter.value('processors')['count'] * 2 ).to_s
  puts "Set $EIR_CORES environment variable to #{ ENV['EIR_CORES'] }".blue

  ENV['EIR_TARGET'] = 'x86_64-pc-linux-gnu'
  puts "Set $EIR_TARGET environment variable to #{ ENV['EIR_TARGET'] }".blue

  ENV['EIR_TARGET32'] = 'i686-pc-linux-gnu'
  puts "Set $EIR_TARGET32 environment variable to #{ ENV['EIR_TARGET32'] }".blue
  
  ENV['EIR_HOST'] = 'x86_64-cross-gnu'
  puts "Set $EIR_HOST environment variable to #{ ENV['EIR_HOST'] }".blue

  ENV['LC_ALL'] = 'POSIX'
  puts "Set locale to #{ ENV['LC_ALL'] }".blue

end

task :default => [ :env, "build/", "stamps/", "output/", "source/", :prepare_packages, :build_toolchain ]
