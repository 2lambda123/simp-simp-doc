#!/usr/bin/rake -T

require 'simp/rake'
require 'yaml'
require 'find'

desc 'Munge Prep'
  desc <<-EOM
This task munges the local working directory and Github
to determine what version and release of the docs should be built.
The version and release are written to build/rpm_metadata/release.
If no defaults can be found, the version defaults to 5.1.X-X.
EOM
task 'munge:prep' do
  # Defaults
  #
  # Location of the release metadata file which is referenced
  # by lua simp-doc.spec
  rel_file = './build/rpm_metadata/release'
  #
  # Location of the simp spec file, which is referenced
  # for default version and release.
  specfile = '../../src/build/simp.spec'
  #
  # Default version and release to ensure we build *something*
  default_simp_version = '5.1.X'
  default_simp_release = 'X'
  #
  # Default header to be written to release metadata file
  release_content = <<-EOM
# DO NOT CHECK IN MODIFIED VERSIONS OF THIS FILE!
# It is automatically generated by rake munge:prep
version: __VERSION__
release: __RELEASE__
EOM

  tmpspec = Tempfile.new('docspec')

  begin

    # Create the release metadata file
    FileUtils.mkdir_p('./build/rpm_metadata')
    unless File.exist?(rel_file)
      fh = File.open(rel_file,"w")
      fh.puts(release_content)
      fh.close
    end

    # If the default specfile does not exist, we need to check Github for a usable spec.
    unless File.exist?(specfile)
      warn("WARNING: No suitable spec file found at #{specfile}, defaulting to https://raw.githubusercontent.com/simp/simp-core/ENV['SIMP_VERSION']/src/build/simp.spec")
      require 'open-uri'

      simp_version = ENV['SIMP_VERSION']

      unless simp_version
        warn("WARNING: Found no ENV['SIMP_VERSION'] set, defaulting to #{default_simp_version}")
        simp_version = default_simp_version
      end

      spec_url = "https://raw.githubusercontent.com/simp/simp-core/#{simp_version}/src/build/simp.spec"
      begin
        open(spec_url) do |specfile|
          tmpspec.write(specfile.read)
        end
      rescue Exception => e
        $stderr.puts e.message
        raise("Could not find a valid spec file at #{spec_url}, check your SIMP_VERSION environment setting!")
      end

      specfile = tmpspec.path
    end

    # Grab the version and release  out of whatever spec we have found.
    begin
      specinfo = Simp::RPM.get_info(specfile)
      simp_version = specinfo[:version]
      simp_release = specinfo[:release]
    rescue Exception
      warn("Could not obtain valid version/release information from #{specfile}, please check for consistency.  Defaulting to #{default_simp_version}-#{default_simp_release}")
      simp_version = default_simp_version
      simp_release = default_simp_release
    end

    # Set the version and release in the rel_file
    %x(sed -i s/version:.*/version:#{simp_version}/ #{rel_file})
    %x(sed -i s/release:.*/release:#{simp_release}/ #{rel_file})
  ensure
    tmpspec.close
    tmpspec.unlink
  end
end

class DocPkg < Simp::Rake::Pkg
  # We need to inject the SCL Python repos for RHEL6 here if necessary
  def mock_pre_check(chroot, *args)
    mock_cmd = super(chroot, *args)

    rh_version = %x(#{mock_cmd} -r #{chroot} -q --chroot 'cat /etc/redhat-release | cut -f3 -d" " | cut -f1 -d"."').chomp

    # This is super fragile
    if rh_version.to_i == 6
      python_repo = 'rhscl-python27-epel-6-x86_64'

      puts "NOTICE: You can ignore any errors relating to RPM commands that don't result in failure"
      %x(#{mock_cmd} -q -r #{chroot} --chroot 'rpmdb --rebuilddb')
      sh  %(#{mock_cmd} -q -r #{chroot} --chroot 'rpm --quiet -q yum') do |ok,res|
        unless ok
          %x(#{mock_cmd} -q -r #{chroot} --install yum)
        end
      end

      sh %(#{mock_cmd} -q -r #{chroot} --chroot 'rpm --quiet -q #{python_repo}') do |ok,res|
        unless ok
          %x(#{mock_cmd} -q -r #{chroot} --install 'https://www.softwarecollections.org/en/scls/rhscl/python27/epel-6-x86_64/download/#{python_repo}.noarch.rpm')
        end
      end

      sh %(#{mock_cmd} -q -r #{chroot} --chroot 'rpm --quiet -q python27') do |ok,res|
        unless ok
          # Fun Fact: Mock (sometimes) adds its default repos to /etc/yum/yum.conf and ignores anything in yum.repos.d
          puts %x(#{mock_cmd} -q -r #{chroot} --chroot 'cat /etc/yum.repos.d/#{python_repo}.repo >> /etc/yum/yum.conf && yum install -qy python27')
        end
      end
    end

    mock_cmd
  end

  def define_clean
    task :clean do
      find_erb_files.each do |erb|
        short_name = "#{File.dirname(erb)}/#{File.basename(erb,'.erb')}"
        if File.exist?(short_name) then
          rm(short_name)
        end
      end
    end
  end

  def define_pkg_tar
    # First, we need to assemble the documentation.
    # This doesn't work properly under Ruby >= 1.9 and leaves cruft in the directories
    # We can try again when 'puppet strings' hits the ground
    super
  end

  def find_erb_files(dir=@base_dir)
    to_ret = []
    Find.find(dir) do |erb|
      if erb =~ /\.erb$/ then
        to_ret << erb
      end
    end

    to_ret
  end
end

DocPkg.new( File.dirname( __FILE__ ) ) do |t|
  # Not sure this is right
  t.clean_list << "#{t.base_dir}/html"
  t.clean_list << "#{t.base_dir}/html-single"
  t.clean_list << "#{t.base_dir}/pdf"
  t.clean_list << "#{t.base_dir}/sphinx_cache"
  t.clean_list << "#{t.base_dir}/docs/dynamic"

  t.exclude_list << 'dist'
  # Need to ignore any generated files from ERB's.
  #t.ignore_changes_list += find_erb_files.map{|x| x = "#{File.dirname(x)}/#{File.basename(x,'.erb')}".sub(/^\.\//,'')}

  Dir.glob('build/rake_helpers/*.rake').each do |helper|
    load helper
  end
end

def process_rpm_yaml(rel,simp_version)
  fail("Must pass release to 'process_rpm_yaml'") unless rel

      header = <<-EOM
SIMP #{simp_version} #{rel} External RPMs
-----------------------------------------

This provides a list of RPMs, and their sources, for non-SIMP components that
are required for system functionality and are specific to an installation on a
#{rel} system.

      EOM

  rpm_data = Dir.glob("../../build/yum_data/SIMP*#{rel}*/packages.yaml")

  table = [ header ]
  table << ".. list-table:: SIMP #{simp_version} #{rel} External RPMs"
  table << '   :widths: 20 80'
  table << '   :header-rows: 1'
  table << ''
  table << '   * - RPM Name'
  table << '     - RPM Source'

  data = ['   * - Not Found']
  data << ['     - Unknown']
  unless rpm_data.empty?
    data = YAML.load_file(rpm_data.sort_by{|filename| File.mtime(filename)}.last)
    data = data.values.map{|x| x = "   * - #{x[:rpm_name]}\n     - #{x[:source]}"}
  end

  table += data

  fh = File.open(File.join('docs','user_guide','RPM_Lists',%(External_SIMP_#{simp_version}_#{rel}_RPM_List.rst)),'w')
  fh.puts(table.join("\n"))
  fh.sync
  fh.close
end

namespace :docs do
  namespace :rpm do
    desc 'Update the RPM lists'
    task :external do
      simp_version = Simp::RPM.get_info('build/simp-doc.spec')[:version]
      ['RHEL','CentOS'].each do |rel|
        process_rpm_yaml(rel,simp_version)
      end
    end

    desc 'Update the SIMP RPM list'
    task :simp do
      simp_version = Simp::RPM.get_info('build/simp-doc.spec')[:version]
      collected_data = []

      if File.directory?('../../src/build')
        default_metadata = YAML.load_file('../../src/build/package_metadata_defaults.yaml')

        Find.find('../../') do |path|
          path_basename = File.basename(path)

          # Ignore hidden directories
          unless path_basename == '..'
            Find.prune if path_basename[0].chr == '.'
          end

          # Ignore spec tests
          Find.prune if path_basename == 'spec'

          # Only Directories
          Find.prune unless File.directory?(path)
          # Ignore symlinks (this may be redundant on some systems)
          Find.prune if File.symlink?(path)

          build_dir = File.join(path,'build')
          if File.directory?(build_dir)
            Dir.chdir(path) do
              # Update the metadata for this RPM
              rpm_metadata = default_metadata.dup
              if File.exist?('build/package_metadata.yaml')
                rpm_metadata.merge!(YAML.load_file('build/package_metadata.yaml'))
              end

              valid_rpm = false
              Array(rpm_metadata['valid_versions']).each do |version_regex|
                if Regexp.new("^#{version_regex}$").match(simp_version)
                  valid_rpm = true
                  break
                end
              end

              if valid_rpm
                # Use the real RPMs if we have them
                rpms = Dir.glob('dist/*.rpm')
                rpms.delete_if{|x| x =~ /\.src\.rpm$/}
                if rpms.empty?
                  Dir.glob('build/*.spec').each do |rpm_spec|
                    pkginfo = Simp::RPM.get_info(rpm_spec)
                    pkginfo[:metadata] = rpm_metadata
                    collected_data << pkginfo
                  end
                else
                  rpms.each do |rpm|
                    pkginfo = Simp::RPM.get_info(rpm)
                    pkginfo[:metadata] = rpm_metadata
                    collected_data << pkginfo
                  end
                end
              end
            end
          end
        end
      end

      header = <<-EOM
SIMP #{simp_version} RPMs
-------------------------

This provides a comprehensive list of all SIMP RPMs and related metadata. Most
importantly, it provides a list of which modules are installed by default and
which are simply available in the repository.

      EOM

      table = [ header ]
      table << ".. list-table:: SIMP #{simp_version} RPMs"
      table << '   :widths: 30 30 30'
      table << '   :header-rows: 1'
      table << ''
      table << '   * - Name'
      table << '     - Version'
      table << '     - Optional'

      data = ['   * - Unknown']
      data << ['     - Unknown']
      data << ['     - Unknown']
      unless collected_data.empty?
        data = collected_data.sort_by{|x| x[:name]}.map{|x| x = "   * - #{x[:name]}\n     - #{x[:full_version]}\n     - #{!x[:metadata]['optional']}"}
      end

      table += data

      fh = File.open(File.join('docs','user_guide','RPM_Lists',"Core_SIMP_#{simp_version}_RPM_List.rst"),'w')
      # Highlight those items that are always there
      fh.puts(table.join("\n").gsub(',true',',**true**'))
      fh.sync
      fh.close
    end
  end

  desc 'build HTML docs'
  task :html do
    extra_args = ''
    ### TODO: decide how we want this task to work
    ### version = File.open('build/simp-doc.spec','r').readlines.select{|x| x =~ /^%define simp_major_version/}.first.chomp.split(' ').last
    ### extra_args = "-t simp_#{version}" if version
    cmd = "sphinx-build -E -n #{extra_args} -b html -d sphinx_cache docs html"
    puts "== #{cmd}"
    %x(#{cmd} > /dev/null)
  end

  desc 'build HTML docs (single page)'
  task :singlehtml do
    extra_args = ''
    cmd = "sphinx-build -E -n #{extra_args} -b singlehtml -d sphinx_cache docs html-single"
    puts "== #{cmd}"
    %x(#{cmd} > /dev/null)
  end

  desc 'build Sphinx PDF docs using the RTD resources (SLOWEST) TODO: BROKEN'
  task :sphinxpdf do
    [ "sphinx-build -E -n -b latex -D language=en -d sphinx_cache docs latex",
      "pdflatex -interaction=nonstopmode -halt-on-error ./latex/*.tex"
    ].each do |cmd|
      puts "== #{cmd}"
      %x(#{cmd} > /dev/null)
    end
  end

  desc 'build PDF docs (SLOWEST)'
  task :pdf do
    extra_args = ''
    cmd = "sphinx-build -T -E -n #{extra_args} -b pdf -d sphinx_cache docs pdf"
    puts "== #{cmd}"
    %x(#{cmd} > /dev/null)
  end

  desc 'Check for broken external links'
  task :linkcheck do
    cmd = "sphinx-build -T -E -n -b linkcheck -d sphinx_cache docs linkcheck"
    puts "== #{cmd}"
    %x(#{cmd})
  end

  desc 'run a local web server to view HTML docs on http://localhost:5000'
  task :server, [:port] => [:html] do |_t, args|
    port = args.to_hash.fetch(:port, 5000)
    puts "running web server on http://localhost:#{port}"
    %x(ruby -run -e httpd html/ -p #{port})
  end
end

# We want to prep for build if possible, but not when running `rake -T`, etc...
Rake.application.tasks.select{|task| task.name.start_with?('docs:', 'pkg:')}.each do |task|
  task.enhance ['munge:prep'] do
  end
end
