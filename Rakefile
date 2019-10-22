require 'erb'
require 'pathname'
require 'open-uri'
require 'date'

JRUBY_VERSIONS = %w{9.1.17.0 9.2.8.0}
JRUBY_NIGHTLY = %{9.2.9.0-SNAPSHOT}
JVM_VERSIONS = %w{8-jdk-slim 11-jdk-slim 13-jdk-slim}

module JRubyDockerfile
  module ClassMethods
    def shas
      (@shas ||= {})
    end
  end

  def self.included(base)
    base.send(:extend, ClassMethods)
    base.send(:attr_reader, :jruby_version, :jvm_version)
  end

  def initialize(jruby_version, jvm_version)
    @jruby_version, @jvm_version, @tag = jruby_version, jvm_version
  end

  def fetch_sha
    self.class.shas[jruby_version] || self.class.shas[jruby_version] = version_sha
  end

  def download_url
    raise NotImplementedError
  end

  def tag
    raise NotImplementedError
  end

  def docker_image_tag
    "#{jruby_version}-#{tag}"
  end

  def sha_url
    "#{download_url}.#{sha_algo}"
  end

  def variant_dir
    Pathname.new("jruby/#{jruby_version}/#{tag}")
  end

  def template
    @template ||= ERB.new(File.read('jruby/Dockerfile.erb', encoding: 'utf-8'))
  end

  def dockerfile_path
    variant_dir.join('Dockerfile')
  end

  def dockerfile_contents
    template.result_with_hash({
      jruby_sha: fetch_sha,
      jvm_version: jvm_version,
      shachecker: "#{sha_algo}sum",
      jruby_url: download_url
    })
  end

  def write_dockerfile
    dockerfile_path.open('w:utf-8') do |f|
      f.write(dockerfile_contents)
    end
  end

  private

  def url_base
    raise NotImplementedError
  end

  def sha_algo
    raise NotImplementedError
  end

  def version_sha
    $stderr.puts "Fetching SHA from #{sha_url}"
    open(sha_url).read.strip
  end

  def download_dist_root
    "https://#{url_base}/org/jruby/jruby-dist/#{jruby_version}"
  end
end

class JRubyStableDockerfile
  include JRubyDockerfile

  def self.variants(jruby_versions, jvm_versions)
    jruby_versions.product(jvm_versions).map { |jruby_version, jvm_version|
      new(jruby_version, jvm_version)
    }
  end

  def download_url
    "#{download_dist_root}/jruby-dist-#{jruby_version}-bin.tar.gz"
  end

  def tag
    jvm_version
  end

  private

  def url_base
    'repo1.maven.org/maven2'
  end

  def sha_algo
    'sha256'
  end
end

class JRubyNightlyDockerfile
  include JRubyDockerfile

  def self.variants(nightly_version, jvm_versions)
    jvm_versions.map { |jvm_version|
      new(nightly_version, jvm_version)
    }
  end

  def self.snapshots
    @snapshots ||= {}
  end

  def download_url
    "#{download_dist_root}/#{most_recent_snapshot}"
  end

  def tag
    "#{snapshot_tag}-#{jvm_version}"
  end

  private

  def url_base
    'oss.sonatype.org/content/repositories/snapshots'
  end

  def sha_algo
    'sha1'
  end

  def most_recent_snapshot
    return self.class.snapshots[jruby_version] if self.class.snapshots.has_key?(jruby_version)
    self.class.snapshots[jruby_version] = begin
      $stderr.puts download_dist_root
      open(download_dist_root + "/").read
        .scan(%r{href="[^"]+#{jruby_version}/(jruby-dist-[^"]+-bin.tar.gz)"})
        .flatten.sort.reverse.first
    end
  end

  def snapshot_tag
    most_recent_snapshot.sub(/^jruby-dist-[0-9\.]+-([0-9]+\.[0-9]+-[0-9]+)-bin.tar.gz$/, '\1').tr('.-', '_')
  end
end

STABLE_VARIANTS = JRubyStableDockerfile.variants(JRUBY_VERSIONS, JVM_VERSIONS)
NIGHTLY_VARIANTS = JRubyNightlyDockerfile.variants(JRUBY_NIGHTLY, JVM_VERSIONS)
VARIANTS = STABLE_VARIANTS + NIGHTLY_VARIANTS

namespace :jruby do
  desc "Remove all generated JRuby Dockerfiles"
  task :clean do
    rm_rf JRUBY_VERSIONS.map { |v, _| "jruby/#{v}" } << "jruby/#{JRUBY_NIGHTLY}"
  end
end

VARIANTS.each do |variant|
  directory variant.variant_dir
  desc "Generate Dockerfile for #{variant.docker_image_tag}"
  file variant.dockerfile_path => [variant.variant_dir, 'jruby/Dockerfile.erb'] do
    variant.write_dockerfile
  end

  desc "Build Docker image for jruby:#{variant.docker_image_tag}"
  task "jruby:#{variant.docker_image_tag}:build" => variant.dockerfile_path do
    sh %{docker build -t fidothe/jruby:#{variant.docker_image_tag} "#{variant.variant_dir}"}
  end

  desc "Push built Docker image for jruby:#{variant.docker_image_tag} to Docker Hub"
  task "jruby:#{variant.docker_image_tag}:push" => "jruby:#{variant.docker_image_tag}:build" do
    sh "docker push fidothe/jruby:#{variant.docker_image_tag}"
  end
end

desc "Generate all Dockerfiles for JRuby"
task :jruby => VARIANTS.map(&:dockerfile_path)

desc "Build all Docker images for JRuby"
task "jruby:build" => VARIANTS.map { |variant| "jruby:#{variant.docker_image_tag}:build" }

desc "Push all Docker images for JRuby"
task "jruby:push" => VARIANTS.map { |variant| "jruby:#{variant.docker_image_tag}:push" }

desc "Remove all generated Dockerfiles"
task :clean => "jruby:clean"

task :default => :jruby