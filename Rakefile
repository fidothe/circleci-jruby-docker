require 'erb'
require 'pathname'
require 'open-uri'
require 'date'

JRUBY_VERSIONS = [
  %w{9.1.17.0         sha256 repo1.maven.org/maven2},
  %w{9.2.8.0          sha256 repo1.maven.org/maven2},
  %w{9.2.9.0-SNAPSHOT sha1   oss.sonatype.org/content/repositories/snapshots}
]
JVM_VERSIONS = %w{8-jdk-slim 11-jdk-slim 13-jdk-slim}

class JRubySHAFetcher
  def self.fetcher
    @fetcher ||= new
  end

  def self.download_url(*args)
    fetcher.download_url(*args)
  end

  def self.fetch_sha(*args)
    fetcher.fetch_sha(*args)
  end

  def initialize
    @shas = {}
    @download_urls = {}
  end

  def fetch_sha(version, sha_algo, url_base)
    key = [version, sha_algo, url_base].join('-')
    @shas[key] || @shas[key] = version_sha(version, sha_algo, url_base)
  end

  def download_url(url_base, version)
    key = [url_base, version].join('-')
    return @download_urls[key] if @download_urls.has_key?(key)
    version_filename = case url_base
      when 'oss.sonatype.org/content/repositories/snapshots'
        $stderr.puts most_recent_snapshot(url_base, version)
        most_recent_snapshot(url_base, version)
      else
        "jruby-dist-#{version}-bin.tar.gz"
      end
    @download_urls[key] = "#{download_dist_root(url_base, version)}#{version_filename}"
  end

  def sha_url(url_base, version, sha_algo)
    "#{download_url(url_base, version)}.#{sha_algo}"
  end

  private

  def version_sha(url_base, version, sha_algo)
    $stderr.puts "Fetching SHA from #{sha_url(url_base, version, sha_algo)}"
    open(sha_url(url_base, version, sha_algo)).read.strip
  end

  def download_dist_root(url_base, version)
    "https://#{url_base}/org/jruby/jruby-dist/#{version}/"
  end

  def most_recent_snapshot(url_base, version)
    $stderr.puts download_dist_root(url_base, version)
    open(download_dist_root(url_base, version)).read
      .scan(%r{href="[^"]+#{version}/(jruby-dist-[^"]+-bin.tar.gz)"})
      .flatten.sort.reverse.first
  end
end

namespace :jruby do
  JRUBY_VERSIONS.each do |jruby_version, sha_algo, jruby_url_base|
    namespace jruby_version do
      JVM_VERSIONS.each do |jvm_version|
        variant_dir = "jruby/#{jruby_version}/#{jvm_version}"
        directory variant_dir

        desc "Generate Dockerfile for #{jruby_version}-#{jvm_version}"
        task jvm_version => [variant_dir, 'jruby/Dockerfile.erb'] do
          template = ERB.new(File.read('jruby/Dockerfile.erb', encoding: 'utf-8'))

          dockerfile = template.result_with_hash({
            jruby_sha: JRubySHAFetcher.fetch_sha(jruby_url_base, jruby_version, sha_algo),
            jvm_version: jvm_version,
            shachecker: "#{sha_algo}sum",
            jruby_url: JRubySHAFetcher.download_url(jruby_url_base, jruby_version)
          })

          Pathname.new(variant_dir).join('Dockerfile').open('w:utf-8') do |f|
            f.write(dockerfile)
          end
        end
      end
    end

    desc "Generate all Dockerfiles for #{jruby_version}"
    task jruby_version => JVM_VERSIONS.map { |v| "jruby:#{jruby_version}:#{v}" }
  end

  desc "Remove all generated JRuby Dockerfiles"
  task :clean do
    rm_rf JRUBY_VERSIONS.map { |v, _| "jruby/#{v}" }
  end
end

desc "Generate all Dockerfiles for JRuby"
task :jruby => JRUBY_VERSIONS.map { |v, _| "jruby:#{v}" }

desc "Remove all generated Dockerfiles"
task :clean => "jruby:clean"

task :default => :jruby