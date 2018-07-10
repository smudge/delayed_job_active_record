require "bundler/gem_helper"
Bundler::GemHelper.install_tasks

require "rspec/core/rake_task"

ADAPTERS = %w[mysql2 postgresql sqlite3].freeze

ADAPTERS.each do |adapter|
  desc "Run RSpec code examples for #{adapter} adapter"
  RSpec::Core::RakeTask.new(adapter => "#{adapter}:adapter")

  namespace adapter do
    task :adapter do
      ENV["ADAPTER"] = adapter
    end
  end
end

task :benchmark do
  ENV["SKIP_COVERAGE"] = "1"
  ENV["ADAPTER"] ||= "mysql2"
  require "./spec/helper"
  require "timeout"

  class Event
    def initialize
      @lock = Mutex.new
      @cond = ConditionVariable.new
      @flag = false
    end

    def set
      @lock.synchronize do
        @flag = true
        @cond.broadcast
      end
    end

    def wait
      @lock.synchronize do
        @cond.wait(@lock) unless @flag
      end
    end
  end

  config = YAML.load(File.read("spec/database.yml"))

  puts "workers\tjob_count\tjob_time\tjobs_completed\trun_time\tjobs/sec"
  [100, 1_000, 10_000, 100_000, 1_000_000].each do |job_count|
    [0.01, 0.1, 1.0, 10.0].each do |job_time|
      [1, 10, 100].each do |worker_count|
        # Reset connection pool
        ActiveRecord::Base.remove_connection config[ENV["ADAPTER"]]
        ActiveRecord::Base.establish_connection config[ENV["ADAPTER"]].merge(pool: 101)

        # Set up test case
        Delayed::Job.delete_all
        job_count.times { delay.sleep(job_time) }
        Delayed::Worker.read_ahead = worker_count
        workers = Array.new(worker_count) { Delayed::Worker.new(quiet: true) }

        # Set up threads
        ready_connections = Queue.new
        start_work = Event.new
        threads = workers.map do |worker|
          Thread.new do
            begin
              ActiveRecord::Base.connection_pool.with_connection do
                ready_connections << self
                start_work.wait
                Timeout.timeout(30) do
                  worker.work_off
                end
              end
            rescue Timeout::Error
              nil
            end
          end
        end

        # Wait for threads to be ready
        loop { break if ready_connections.length == threads.length }

        # Signal to threads to begin work.
        start_time = Time.now
        start_work.set
        threads.map(&:join)
        completion_time = Time.now - start_time

        # Output summary
        finished_jobs = job_count - Delayed::Job.count
        jobs_per_sec = finished_jobs / completion_time
        puts "#{worker_count}\t#{job_count}\t#{job_time}\t#{finished_jobs}\t#{completion_time}\t#{jobs_per_sec}"
      end
    end
  end
end

task :coverage do
  ENV["COVERAGE"] = "true"
end

task :adapter do
  ENV["ADAPTER"] = nil
end

Rake::Task[:spec].enhance do
  require "simplecov"
  require "coveralls"

  Coveralls::SimpleCov::Formatter.new.format(SimpleCov.result)
end

require "rubocop/rake_task"
RuboCop::RakeTask.new

task default: ([:coverage] + ADAPTERS + [:adapter] + [:rubocop])
