DB_PARENT_DIR = ENV.fetch('DB_PARENT_DIR', '/tmp')
OUTPUT_PARENT_DIR = ENV.fetch('OUTPUT_PARENT_DIR', '/tmp/html')
QUIET = ENV.fetch('QUIET', false)

OLD_VERSIONS = %w[1.8.7 1.9.3 2.0.0 2.1.0 2.2.0]
SUPPORTED_VERSIONS = %w[2.3.0 2.4.0 2.5.0]
UNRELEASED_VERSIONS = %w[2.6.0]
ALL_VERSIONS = [*OLD_VERSIONS, *SUPPORTED_VERSIONS, *UNRELEASED_VERSIONS]

def generate_database(version)
  puts "generate database of #{version}"
  db = "#{DB_PARENT_DIR}/db-#{version}"
  succeeded = system("bundle", "exec",
                     "bitclust", "--database=#{db}",
                     "init", "version=#{version}", "encoding=UTF-8")
  raise "Failed to initialize BitClust database" unless succeeded
  succeeded = system("bundle", "exec", "bitclust", "--database=#{db}",
                     "update", "--stdlibtree=refm/api/src")
  raise "Failed to update BitClust database" unless succeeded
  capi_files = Dir.glob("refm/capi/src/*")
  succeeded = system("bundle", "exec", "bitclust", "--database=#{db}", "--capi",
                     "update", *capi_files)
  raise "Failed to update BitClust C API database" unless succeeded
end

def generate_statichtml(version)
  db = "#{DB_PARENT_DIR}/db-#{version}"
  generate_database(version) unless File.exist?(db)
  puts "generate static html of #{version}"
  outputdir = "#{OUTPUT_PARENT_DIR}/#{version}"
  bitclust_gem_path = File.expand_path('../..', `bundle exec gem which bitclust`)
  raise "bitclust gem not found" unless $?.success?
  succeeded = system("bundle", "exec",
                     "bitclust", "--database=#{db}",
                     "statichtml", "--outputdir=#{outputdir}",
                     "--templatedir=#{bitclust_gem_path}/data/bitclust/template.offline",
                     "--catalog=#{bitclust_gem_path}/data/bitclust/catalog",
                     "--fs-casesensitive",
	             (QUIET ? "--quiet" : "--no-quiet"),
                     "--canonical-base-url=http://localhost:9292/latest/")
  raise "Failed to generate static html" unless succeeded
  File.unlink("#{OUTPUT_PARENT_DIR}/latest") rescue nil
  File.symlink(version, "#{OUTPUT_PARENT_DIR}/latest")
end

task :default => [:generate, :check_prev_commit_format]

namespace :generate do
  ALL_VERSIONS.each do |version|
    desc "Generate document database of #{version}"
    task version do
      generate_database(version)
    end
  end

  desc "Generate document database of all versions"
  task :all => ALL_VERSIONS

  desc "Generate document database for old versions"
  task :old => OLD_VERSIONS

  desc "Generate document database for supported versions"
  task :supported => SUPPORTED_VERSIONS

  desc "Generate document database for unreleased versions"
  task :unreleased => UNRELEASED_VERSIONS
end

desc "Generate document database"
task :generate => [*OLD_VERSIONS, *SUPPORTED_VERSIONS].map {|version| "generate:#{version}" }

namespace :statichtml do
  ALL_VERSIONS.each do |version|
    desc "Generate static html of #{version}"
    task version do
      generate_statichtml(version)
    end
  end

  desc "Generate static html of all versions"
  task :all => ALL_VERSIONS

  desc "Generate static html for old versions"
  task :old => OLD_VERSIONS

  desc "Generate static html for supported versions"
  task :supported => SUPPORTED_VERSIONS

  desc "Generate static html for unreleased versions"
  task :unreleased => UNRELEASED_VERSIONS
end

desc "Generate static html"
task :statichtml => [*OLD_VERSIONS, *SUPPORTED_VERSIONS].map {|version| "statichtml:#{version}" }

desc "Check previous commit format"
task :check_prev_commit_format do
  change_files = `git diff HEAD^ HEAD --name-only --diff-filter=d`.split
  ignored_files = %w[
     refm/api/src/_builtin/BasicObject.private_methods_from_Object
     refm/api/src/_builtin/BasicObject.public_methods_from_Object
     refm/api/src/_builtin/Module.alias_method
     refm/api/src/_builtin/Module.attr
     refm/api/src/_builtin/Module.define_method
     refm/api/src/_builtin/Module.include
     refm/api/src/_builtin/Module.prepend
     refm/api/src/_builtin/Module.remove_method
     refm/api/src/_builtin/Module.undef_method
     refm/api/src/_builtin/constants
     refm/api/src/_builtin/functions
     refm/api/src/_builtin/functions_pp
     refm/api/src/_builtin/specialvars
     refm/api/src/rake/core_ext
     refm/api/src/rdoc/RDoc__constants
     refm/api/src/shell/builtincommands
     refm/api/src/socket/constants
     refm/api/src/tkextlib/tkDND/shape.rd
     refm/api/src/tkextlib/tkDND/tkdnd.rd
     refm/api/src/tkextlib/tktrans/tktrans.rd
  ]
  res = []
  [*SUPPORTED_VERSIONS, *UNRELEASED_VERSIONS].each do |v|
    change_files.each do |path|
      if %r!\Arefm/api/!.match(path) && !ignored_files.include?(path)
        htmls = []
        htmls << `bundle exec bitclust htmlfile --ruby=#{v} #{path}`
        raise "Failed to bitclust htmlfile, ruby: #{v}, path: #{path}" unless $?.success?
        File.read(path).scan(/^=([^=]\w+)\s*(\S+)\s*(?:<\s*(\S+))?/).each do |t, k, pk|
          next if /\bre(open|define)\b/.match(t)
          html = `bundle exec bitclust htmlfile --ruby=#{v} --target=#{k} #{path} 2>&1`
          next if /\Abitclust: error: no such/.match(html) && !$?.success?
          raise "Failed to bitclust htmlfile, ruby: #{v}, target: #{k}, path: #{path}" unless $?.success?
          htmls << html

          a = html.lines.grep(/\[UNKNOWN_META_INFO\]/)
          if !a.empty?
            res.push("Found invalid meta info: #{a.first.chomp} in #{v}:#{path}")
          end
        end
        htmls.each do |html|
          a = html.lines.grep(/<span class="compileerror">/)
          if !a.empty?
            res.push("Found invalid format link: #{a.first.chomp} in #{v}:#{path}")
          end
        end
      end
    end
  end
  if res.size > 0
    raise res.sort.uniq.join("\n")
  end
end
