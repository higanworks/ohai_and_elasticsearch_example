# coding: utf-8

require 'fileutils'
require 'webster'
require 'fauxhai'
require 'excon'

ES_URL       = 'https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.1.1.tar.gz'
ES_IMAGE     = File.basename(ES_URL)
KIBANA_URL   = 'https://download.elasticsearch.org/kibana/kibana/kibana-3.1.0.tar.gz'
KIBANA_IMAGE = File.basename(KIBANA_URL)

desc "Print Usage"
task :default do
  # Usage
  system('rake -vT')
end

desc 'start example'
task :up do
  Rake::Task['prepare:all'].invoke
  Rake::Task['ps:es:start'].invoke
  Rake::Task['ps:es:wait_for'].invoke
  Rake::Task['mock:put'].invoke
  Rake::Task['ps:kibana:start'].invoke
end

desc 'stop example'
task :down do
  Rake::Task['ps:es:stop'].invoke
end


namespace :prepare do
  desc 'prepare softwares for development'
  task :all do
    puts 'Prepare elasticsearch and kibana..'
    Rake::Task['prepare:es'].invoke
    Rake::Task['prepare:kibana'].invoke
  end

  desc 'setup elasticsearch'
  task :es do
    unless File.exists?('elasticsearch')
      system("wget #{ES_URL}") unless File.exists?(ES_IMAGE)
      system("tar xvzf #{ES_IMAGE}")
      FileUtils.move File.basename(ES_IMAGE, '.tar.gz'), 'elasticsearch'
    end
  end

  desc 'setup kibana'
  task :kibana do
    unless File.exists?('kibana')
      system("wget #{KIBANA_URL}") unless File.exists?(KIBANA_IMAGE)
      system("tar xvzf #{KIBANA_IMAGE}")
      FileUtils.move File.basename(KIBANA_IMAGE, '.tar.gz'), 'kibana'
    end
  end
end

namespace :ps do
  namespace :es do
    desc 'start elasticsearch'
    task :start do
      if File.exists?('tmp/es.pid')
        puts 'FATAL: pid file already exists !!'
        puts 'FATAL: Start canceled'
      else
        puts 'Start elasticsearch !!'
        system('elasticsearch/bin/elasticsearch -d -p tmp/es.pid')
      end
    end

    task :wait_for do
      NodeHelper.wait_for_avail
    end

    desc 'stop elasticsearch'
    task :stop do
      if File.exists?('tmp/es.pid')
        puts 'Stop elasticsearch...'
        system('bash -c "kill `cat tmp/es.pid`"')
      end
    end
  end

  namespace :kibana do
    desc 'start python web server default port=8000'
    task :start, :port
    task :start do |x, args|
      cmd = ['python -mSimpleHTTPServer']
      cmd << args[:port] unless args[:port].to_i == 0
      system("cd kibana; #{cmd.join(' ')}")
    end
  end
end

namespace :mock do
  desc 'show sample ohai'
  task :test do
    puts JSON.pretty_generate NodeHelper.get_mock
  end

  desc 'put ohai samples to elasticsearch defaut num=30'
  task :put, :num
  task :put do |x, args|
    if args[:num].to_i == 0
      num = 30
    else
      num = args[:num].to_i
    end

    puts "put #{num} ohai mocks to elasticsearch"
    num.times do
      mock = NodeHelper.get_mock
      NodeHelper.post_like_curl(mock)
    end
  end
end

module NodeHelper
  @webster = Webster.new
  @es_endpoint = ENV['ES_ENDPOINT'] || 'http://localhost:9200'
  @connection = Excon.new(@es_endpoint)

  def self.get_mock
    platforms = Dir.glob(File.join(Fauxhai.root, 'lib', 'fauxhai', 'platforms', '*')).map {|p| File.basename p}
    platform = platforms.sample
    versions = Dir.glob(File.join(Fauxhai.root, 'lib', 'fauxhai', 'platforms', platform, '*.json')).map {|v| File.basename v, '.json'}
    mock = Fauxhai.mock(platform: platform, version: versions.sample) do |node|
      host_name = "#{@webster.random_word}.example.com"
      node['hostname'] = host_name
      node['fqdn'] = host_name
    end
    mock.data
  end

  def self.post_like_curl(node)
    puts "Example: 'ohai | curl -XPUT #{@es_endpoint}/ohai/node/#{node['fqdn']} -d @-' => #{node['platform']}"
    @connection.put(path: "/ohai/node/#{node['fqdn']}", body: node.to_json)
  end

  def self.wait_for_avail
    sleep 2
    begin
      until @connection.get.status == 200
        sleep 2
      end
    rescue
      puts 'Waiting es available....'
      sleep 2
      wait_for_avail
    end
  end
end
