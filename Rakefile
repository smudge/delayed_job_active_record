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
  ENV["ADAPTER"] = "mysql2"
  require "./spec/helper"
  require "benchmark"

  Benchmark.bm(80) do |x|
    [100, 1_000, 10_000, 100_000].each do |job_count|
      [0.1, 1.0, 10.0].each do |job_time|
        [1, 10, 100, 1_000].each do |worker_count|
          next if (job_time * job_count) / worker_count > 600 # skip cases that will optimally take more than 10 minutes

          Delayed::Job.delete_all
          job_count.times { delay.sleep(job_time) }
          work_off = (job_count.to_f / worker_count).ceil
          Delayed::Worker.read_ahead = worker_count
          workers = Array.new(worker_count) { Delayed::Worker.new(quiet: true) }

          x.report("workers: #{worker_count}, job_count: #{job_count}, job_time: #{job_time}") do
            workers.map { |worker| Thread.new { worker.work_off(work_off) } }.map(&:join)
          end
        end
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
