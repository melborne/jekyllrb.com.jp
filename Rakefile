require "octokit"
require "dotenv"

Dotenv.load

GITHUB_USER = 'jekyll'
GITHUB_REPOSITORY = 'jekyll'
RAW_URL = 'https://raw.githubusercontent.com'

BASE_REVISION = '5daf987'

task :default => :togglate

desc "diff local and original document at BASE_REVISION"
# args:
#   rev: base rivision(ex: rev=master)
#   files: diff target file(ex: files=docs/*.md )
task :togglate do
  revision = ENV['rev']
  if revision.nil?
    revision = BASE_REVISION
  end
  puts "base revision: #{revision}"

  ORIGINAL_DOC_URL = "#{RAW_URL}/#{GITHUB_USER}/#{GITHUB_REPOSITORY}/#{revision}/site"
  puts "original doc url: #{ORIGINAL_DOC_URL}"

  files = ENV['files']
  if files.nil?
    files = `git diff --name-only HEAD~ -- docs`
    files = files.split("\n")
  end
  puts "files: #{files}"

  ok_files = []
  ng_files = []

  Dir.glob(files).each do |file|
    begin
      # check togglate command
      togglate = 'togglate'
      system( "#{togglate} > /dev/null" )
      unless $? == 0
        togglate = "bundle exec #{togglate}"
        system ( "#{togglate} > /dev/null" )
        unless $? == 0
          puts 'Please install togglate: $ gem install togglate'
          exit 1
        end
      end

      local_doc = "#{file}_local"
      origin_doc = "#{file}_origin"

      # output local document
      system( "#{togglate} commentout #{file} > #{local_doc}" )
      # output original document
      file_exist = system( "curl -sf #{ORIGINAL_DOC_URL}/#{file} > #{origin_doc}" )
      unless file_exist
        fail "`curl`: No such file - '#{ORIGINAL_DOC_URL}/#{file}'"
      end
      # system( "git diff --no-index #{local_doc} #{origin_doc}" )
      system( "diff -u #{local_doc} #{origin_doc}" )
      case $?
      when 0
        ok_files << file
      else
        ng_files << file
      end
    ensure
      [local_doc, origin_doc].each do |f|
        system("rm", f) if File.exist?(f)
      end
    end
  end

  puts "OK:"
  p ok_files
  puts "NG:"
  p ng_files

  if ng_files.empty?
    exit 0
  else
    exit 1
  end
end

desc "compare file exsistence with site/docs in original BASE_REVISION"
# args:
#   rev: base rivision(ex: rev=master) default: master
task :compare_docs do
  revision = ENV['rev'] || 'master'

  local_files = Dir.glob('docs/*').map { |f| File.basename(f) }
  remote_files = Octokit.contents("#{GITHUB_USER}/#{GITHUB_REPOSITORY}", path:'site/docs', ref:"#{revision}").map(&:name)

  added_files = remote_files - local_files
  removed_files = local_files - remote_files

  case
  when [added_files, removed_files].all?(&:empty?)
    # say nothing
  else
    puts "New files: #{added_files.join(', ')}"
    puts "Removed files: #{removed_files.join(', ')}"
  end
end

desc "jekyll syntax check (try jekyll build)"
task :jekyll do
  # check jekyll command
  jekyll = 'jekyll'
  system( "#{jekyll} > /dev/null" )
  unless $? == 0
    jekyll = "bundle exec #{jekyll}"
    system ( "#{jekyll} > /dev/null" )
    unless $? == 0
      puts 'Please install jekyll: $ gem install jekyll'
      exit 1
    end
  end

  # create _config.yml
  config_yml = '_config_rake_jekyll.yml'
  system( "echo 'exclude: ['vendor']' >> #{config_yml}" )
  system( "echo 'markdown: kramdown' >> #{config_yml}" )

  # try jekyll build
  system( "#{jekyll} build --config _config.yml,#{config_yml}" )
  result = $?

  system( "rm #{config_yml}" )

  case result
  when 0
    exit 0
  else
    exit 1
  end
end

desc "Create issue for a file"
task :create_issue, "path"
task :create_issue do |x, args|
  path = args.path || fail("A file path required: e.g. create_issue['diff/docs/index.diff']")

  host = "https://github.com"
  myrepo = ENV['MYREPO']
  myrevision = ENV['MYREVISION']

  title, body, label =
    case path
    when /diff$/
      build_update_issue_content(host, myrepo, myrevision, path)
    when /(md|markdown)$/
      # build_translate_issue_content(host, myrepo, myrevision, path)
    else
      fail "Only accept 'diff' or 'markdown'"
    end

  Octokit.configure { |c| c.access_token = ENV['TOKEN'] }
  Octokit.create_issue(myrepo, title, body, labels:label)
  puts "Issue created successfully for #{path}"
  exit(0)
end

def build_update_issue_content(host, repo, revision, path)
  repo_diff_file_link = File.join(host, repo, 'blob', revision, path)
  repo_md_file_link = build_repo_file_link(host, repo, revision, path, '.md')
  original_rev_link =
    if base_rev = read_base_revision(path)
      File.join(host, GITHUB_USER, GITHUB_REPOSITORY, 'commit', base_rev)
    else
      ''
    end
  label = 'Original Updated'
  title = "Need to update! #{File.basename(repo_md_file_link)}"
  body =<<-EOS
Original file revised. Need to update our translation:

  Diff: #{repo_diff_file_link}
  File: #{repo_md_file_link}
  Base Revision: #{original_rev_link}
    EOS
  return title, body, label
end

def build_translate_issue_content(host, repo, revision, path)
  
end

def build_repo_file_link(host, repo, revision, path, ext)
  _, *dir, file = path.split('/')
  file = File.basename(file, '.*') + ext
  File.join(host, repo, "blob", revision, *dir, file)
end

def read_base_revision(path)
  rev_re = /^(?:B|b)ase.revision:\s*([a-z0-9]+)/
  md = File.read(path).match(rev_re)
  md ? md[1] : nil
end
