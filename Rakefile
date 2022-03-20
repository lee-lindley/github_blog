GH_PAGES_DIR = "lee-lindley.github.io"

desc "Build Jekyll site and copy files. remember to exclude Rakefile in _config.yml"
task :build do
    system "bundle exec jekyll build"
    system "rm -r ../#{GH_PAGES_DIR}/*" unless Dir['../#{GH_PAGES_DIR}/*'].empty?
    system "cp -r _site/* ../#{GH_PAGES_DIR}/"
end
