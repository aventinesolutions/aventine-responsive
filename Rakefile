require 'dotenv/load'
require 'net/ftp'
require 'yaml'
require 'pathname'
require 'forwardable'

class FTPClient
  extend Forwardable
  attr_reader :remote_path

  def_delegators :@ftp, :chdir, :delete, :putbinaryfile

  def initialize(remote_path)
    @remote_path = remote_path
  end

  def ftp
    @ftp ||= Net::FTP.new
  end

  def connect
    ftp.connect(ENV['FTP_SERVER'])
    ftp.login(ENV['FTP_USER'], ENV['FTP_PASSWORD'])
    ftp.passive = true
    ftp.debug_mode = ENV['FTP_DEBUG'] ? eval(ENV['FTP_DEBUG']) : false
    ftp.chdir(remote_path)
  end

  def delete_recursive(file_or_dir)
    if file_or_dir == list(file_or_dir).first
      puts "Removing file: #{file_or_dir}"
      ftp.delete(file_or_dir)
    else
      list(file_or_dir).each { |entry| delete_recursive(file_or_dir + "/" + entry) }
      puts "Removing directory: #{file_or_dir}"
      ftp.rmdir(file_or_dir)
    end
  end

  def copy_recursive(file_or_dir, prefix_to_remove = nil)
    remote_file_or_dir = Pathname(prefix_to_remove ? file_or_dir.to_s.gsub(prefix_to_remove, "") : file_or_dir)
    if file_or_dir.directory?
      puts "Creating directory #{remote_file_or_dir}"
      ftp.mkdir(remote_file_or_dir.to_s)
      Pathname.glob(Pathname.new('.').join(file_or_dir, '*')).each do |entry|
        copy_recursive(entry, prefix_to_remove)
      end
    else
      puts "Creating file #{remote_file_or_dir}"
      ftp.putbinaryfile(file_or_dir, remote_file_or_dir.to_s)
    end
  end

  # file list
  def list(path = nil)
    ftp.nlst(path).select { |entry| entry !~ /^\.{1,2}$/ }
  end
end

class Deployer
  def self.run(mappings)
    ftp_client = FTPClient.new($spec['remote_home'])
    ftp_client.connect

    # Remove all files
    begin
      ftp_client.delete($spec['home_page'])
    rescue Net::FTPPermError
      puts 'home page does not exist yet on server'
    end
    mappings.each do |_, remote|
      ftp_client.chdir(remote)
      ftp_client.list.each do |entry|
        ftp_client.delete_recursive(entry)
      end
    end

    # Copy files placed in public directory
    ftp_client.chdir($spec['remote_home'])
    puts 'copying home page to remote server'
    ftp_client.putbinaryfile($spec['home_page'], $spec['home_page'])
    mappings.each do |local, remote|
      ftp_client.chdir(remote)
      Pathname.glob(local + "/*").each do |entry|
        ftp_client.copy_recursive(entry, local + "/")
      end
    end

  ensure
    ftp_client.ftp.close
  end
end

desc 'check environment'
task :check_env do
  errors = []
  %w[SERVER USER PASSWORD].each do |param|
    key = "FTP_#{param}"
    val = ENV[key]
    errors.push("must define environment value for #{key}") unless val && val.length > 0
  end
  fail errors.join(', ') unless errors.length == 0
  puts 'environment is good to go!'
end

desc 'clean, compile and minify'
task :compile do
  puts 'TODO'
end

desc 'read deployment specifications'
task :spec do
  $spec = YAML.load_file('deploy.yml')
end

desc 'deploy the specified local files to the remote server'
task :deploy do
  Deployer.run($spec['folders'])
end

desc 'default: deploy local files to remote server after compiling'
task default: [:check_env, :spec, :compile, :deploy]

desc 'debugging'
task debug: [:check_env, :spec, :deploy]