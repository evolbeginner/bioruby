#
# = Rakefile - helper of developement and packaging
#
# Copyright::   Copyright (C) 2009, 2012 Naohisa Goto <ng@bioruby.org>
# License::     The Ruby License
#

require 'rubygems'
require 'erb'
require 'pathname'
require 'fileutils'
require 'tmpdir'
require 'rake/testtask'
require 'rake/packagetask'

begin
  require 'rubygems/package_task'
rescue LoadError
  # old RubyGems/Rake version
  require 'rake/gempackagetask'
end

begin
  require 'rdoc/task'
rescue LoadError
  # old RDoc/Rake version
  require 'rake/rdoctask'
end

# workaround for new module name
unless defined? Rake::GemPackageTask then
  Rake::GemPackageTask = Gem::PackageTask
end

load "./lib/bio/version.rb"
BIO_VERSION_RB_LOADED = true

# Version string for tar.gz, tar.bz2, or zip archive.
# If nil, use the value in lib/bio.rb
# Note that gem version is always determined from bioruby.gemspec.erb.
version = ENV['BIORUBY_VERSION'] || Bio::BIORUBY_VERSION.join(".")
version = nil if version.to_s.empty?
extraversion = ENV['BIORUBY_EXTRA_VERSION'] || Bio::BIORUBY_EXTRA_VERSION
extraversion = nil if extraversion.to_s.empty?
BIORUBY_VERSION = version
BIORUBY_EXTRA_VERSION = extraversion

task :default => "see-env"

Rake::TestTask.new do |t|
  t.test_files = FileList["test/{unit,functional}/**/test_*.rb"]
end

Rake::TestTask.new do |t|
  t.name = :"test-all"
  t.test_files = FileList["test/{unit,functional,network}/**/test_*.rb"]
end

Rake::TestTask.new do |t|
  t.name = :"test-network"
  t.test_files = FileList["test/network/**/test_*.rb"]
end

# files not included in gem but included in tar archive
tar_additional_files = []

GEM_SPEC_FILE = "bioruby.gemspec"
GEM_SPEC_TEMPLATE_FILE = "bioruby.gemspec.erb"

# gets gem spec string
current_gem_spec_string = File.read(GEM_SPEC_FILE) rescue nil

next_gem_spec_string = File.open(GEM_SPEC_TEMPLATE_FILE, "rb") do |f|
  ERB.new(f.read).result
end

# gets gem spec object
current_spec = eval(current_gem_spec_string || '')
next_spec = eval(next_gem_spec_string)
spec = (current_spec || next_spec)

# adds notice of automatically generated file
next_gem_spec_string = "# This file is automatically generated from #{GEM_SPEC_TEMPLATE_FILE} and\n# should NOT be edited by hand.\n# \n" + next_gem_spec_string

# compares current gemspec file and newly generated gemspec string
if current_gem_spec_string &&
   current_gem_spec_string != next_gem_spec_string then
  #Rake::Task[GEM_SPEC_FILE].invoke
  flag_update_gemspec = true
else
  flag_update_gemspec = false
end

desc "Update gem spec file"
task :gemspec => GEM_SPEC_FILE

desc "Force update gem spec file"
task :regemspec do
  #rm GEM_SPEC_FILE, :force => true
  Rake::Task[GEM_SPEC_FILE].execute(nil)
end

desc "Update #{GEM_SPEC_FILE}"
file GEM_SPEC_FILE => [ GEM_SPEC_TEMPLATE_FILE, 'Rakefile',
                        'lib/bio/version.rb' ] do |t|
  puts "creates #{GEM_SPEC_FILE}"
  File.open(t.name, 'wb') do |w|
    w.print next_gem_spec_string
  end
end

task :package => [ GEM_SPEC_FILE ] do
  Rake::Task[:regemspec].invoke if flag_update_gemspec
end

pkg_dir = "pkg"
tar_version = (BIORUBY_VERSION || spec.version) + BIORUBY_EXTRA_VERSION.to_s
tar_basename = "bioruby-#{tar_version}"
tar_filename = "#{tar_basename}.tar.gz"
tar_pkg_filepath = File.join(pkg_dir, tar_filename)
gem_filename = spec.full_name + ".gem"
gem_pkg_filepath = File.join(pkg_dir, gem_filename)

Rake::PackageTask.new("bioruby") do |pkg|
  #pkg.package_dir = "./pkg"
  pkg.need_tar_gz = true
  pkg.package_files.import(spec.files)
  pkg.package_files.include(*tar_additional_files)
  pkg.version = tar_version
end

Rake::GemPackageTask.new(spec) do |pkg|
  #pkg.package_dir = "./pkg"
end

Rake::RDocTask.new do |r|
  r.rdoc_dir = "rdoc"
  r.rdoc_files.include(*spec.extra_rdoc_files)
  r.rdoc_files.import(spec.files.find_all {|x| /\Alib\/.+\.rb\z/ =~ x})
  #r.rdoc_files.exclude /\.yaml\z"
  opts = spec.rdoc_options.to_a.dup
  if i = opts.index('--main') then
    main = opts[i + 1]
    opts.delete_at(i)
    opts.delete_at(i)
  else
    main = 'README.rdoc'
  end
  r.main = main
  r.options = opts
end

# Tutorial files
TUTORIAL_RD =    'doc/Tutorial.rd'
TUTORIAL_RD_JA = 'doc/Tutorial.rd.ja'

TUTORIAL_RD_HTML    = TUTORIAL_RD    + '.html'
TUTORIAL_RD_JA_HTML = TUTORIAL_RD_JA + '.html'

HTMLFILES_TUTORIAL = [ TUTORIAL_RD_HTML, TUTORIAL_RD_JA_HTML ]

# Formatting RD to html.
def rd2html(src, dst)
  title = File.basename(src)
  sh "rd2 -r rd/rd2html-lib.rb --with-css=bioruby.css --html-title=#{title} #{src} > #{dst}"
end

# Tutorial.rd to Tutorial.rd.html
file TUTORIAL_RD_HTML => TUTORIAL_RD do |t|
  rd2html(t.prerequisites[0], t.name)
end

# Tutorial.rd.ja to Tutorial.html.ja
file TUTORIAL_RD_JA_HTML => TUTORIAL_RD_JA do |t|
  rd2html(t.prerequisites[0], t.name)
end

desc "Update doc/Tutorial*.html"
task :tutorial2html => HTMLFILES_TUTORIAL

desc "Force update doc/Tutorial*.html"
task :retutorial2html do
  # safe_unlink HTMLFILES_TUTORIAL
  HTMLFILES_TUTORIAL.each do |x|
    Rake::Task[x].execute(nil)
  end
end

# ChangeLog
desc "Force update ChangeLog using git log"
task :rechangelog do
  # The tag name in the command line should be changed
  # after releasing new version, updating ChangeLog,
  # and doing "git mv ChangeLog doc/ChangeLog-X.X.X".
  sh "git log --stat --summary 1.5.0..HEAD > ChangeLog"
end

# define mktmpdir
if true then
  # Note: arg is a subset of Dir.mktmpdir
  def mktmpdir(prefix)
    ## prepare temporary directory for testing
    top = Pathname.new(File.join(Dir.pwd, "tmp")).cleanpath.to_s
    begin
      Dir.mkdir(top)
    rescue Errno::EEXIST
    end

    ## prepare working directory
    flag = false
    dirname = nil
    ret = nil
    begin
      10.times do |n|
        # following 3 lines are copied from Ruby 1.9.3's tmpdir.rb and modified
        t = Time.now.strftime("%Y%m%d")
        path = "#{prefix}#{t}-#{$$}-#{rand(0x100000000).to_s(36)}"
        path << "-#{n}" if n > 0
        begin
          dirname = File.join(top, path)
          flag = Dir.mkdir(dirname)
          break if flag
        rescue SystemCallError
        end
      end
      raise "Couldn't create a directory under #{tmp}." unless flag
      ret = yield(dirname)
    ensure
      FileUtils.remove_entry_secure(dirname, true) if flag and dirname
    end
    ret
  end #def mktmpdir
## Currently, Dir.mktmpdir isn't used Because of JRuby's behavior.
elsif Dir.respond_to?(:mktmpdir) then
  def self.mktmpdir(*arg, &block)
    Dir.mktmpdir(*arg, &block)
  end
else
  load "lib/bio/command.rb"
  def mktmpdir(*arg, &block)
    Bio::Command.mktmpdir(*arg, &block)
  end
end

def chdir_with_message(dir)
  $stderr.puts("chdir #{dir}")
  Dir.chdir(dir)
end

# run in different directory
def work_in_another_directory
  pwd = Dir.pwd
  ret = false
  mktmpdir("bioruby") do |dirname|
    begin
      chdir_with_message(dirname)
      ret = yield(dirname)
    ensure
      chdir_with_message(pwd)
    end
  end
  ret
end
    
desc "task specified with BIORUBY_RAKE_DEFAULT_TASK (default \"test\")"
task :"see-env" do
  t = ENV["BIORUBY_RAKE_DEFAULT_TASK"]
  if t then
    Rake::Task[t].invoke
  else
    Rake::Task[:test].invoke
  end
end

desc "test installed bioruby on system"
task :"installed-test" do
  data_path = File.join(Dir.pwd, "test/data")
  test_runner = File.join(Dir.pwd, "test/runner.rb")
  data_path = Pathname.new(data_path).cleanpath.to_s
  test_runner = Pathname.new(test_runner).cleanpath.to_s

  ENV["BIORUBY_TEST_DATA"] = data_path
  ENV["BIORUBY_TEST_LIB"] = ""
  ENV["BIORUBY_TEST_GEM"] = nil

  work_in_another_directory do |dirname|
    ruby("-rbio", test_runner)
  end
end

desc "test installed bioruby gem version #{spec.version.to_s}"
task :"gem-test" do
  data_path = File.join(Dir.pwd, "test/data")
  test_runner = File.join(Dir.pwd, "test/runner.rb")
  data_path = Pathname.new(data_path).cleanpath.to_s
  test_runner = Pathname.new(test_runner).cleanpath.to_s

  ENV["BIORUBY_TEST_DATA"] = data_path
  ENV["BIORUBY_TEST_LIB"] = nil
  ENV["BIORUBY_TEST_GEM"] = spec.version.to_s

  work_in_another_directory do |dirname|
    ruby(test_runner)
  end
end

