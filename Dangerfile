require 'active_support/core_ext/string'
require 'set'

def warn_zero_downtime
  migration_files = (git.added_files - %w(Dangerfile)).grep(%r(db/migrate))
  return nil if migration_files.empty?

  markdown <<~MARKDOWN
    It looks like you have done a migration.
    Beware about [zero downtime](https://medium.com/klaxit-techblog/47528abe5136)!
  MARKDOWN


  if git.modified_files.grep(%r(db/structure.sql|db/schema.rb$)).empty?
    fail("You should commit your databases changes via `structure.sql` or `schema.rb` when your do a migration.")
  end

  modified_tables = migration_files
                    .map { |file| file[/_add_(\w+)_to_(\w+?).rb$/, 2]&.singularize }
                    .compact

  zero_downtime_models = (git.modified_files + git.added_files - %w(Dangerfile))
                         .select { |file| file.match?(%r(app/models/\w+.rb$)) }
                         .select { |file| git.diff_for_file(file).patch.match?(/^\+.*REMOVE ON NEXT RELEASE$/) }
                         .map { |file| file[%r(app/models/(\w+).rb$), 1] }

  no_zero_downtime_models = modified_tables - zero_downtime_models

  return nil if no_zero_downtime_models.empty?

  # DEBUG: uncomment next lines
  # require 'pry'
  # binding.pry

  markdown <<~MARKDOWN
    These tables looks like they are not covered by zero downtime:
    #{no_zero_downtime_models.map { |model| " - [ ] `#{model.camelcase}`" } * "\n"}
  MARKDOWN
end

def warn_rubocop
  rubocop.lint(files: git.modified_files + git.added_files,
               inline_comment: true,
               force_exclusion: true)
end

# helper
def line_number_from_diff(file, line)
  IO.readlines(file).index(line[1..-1] + "\n") + 1
end

def warn_no_spec_for_method
  return fail("`spec` directory is missing") unless Dir.exist?("spec")

  # Ruby files but not specs: https://regex101.com/r/xjQMQX/1
  ruby_file_regex = /^(?!.*_spec\.rb).*\.rb$/
  # https://regex101.com/r/xLymHd/1
  new_method_regex = /^\+\s+def \b(?:self\.)?([^(\s]+).*/
  spec_regex = /describe "[#.]([^"]+)"/
  total_uncovered = 0
  files_to_spec = (git.modified_files + git.added_files)
                  .grep(ruby_file_regex)

  new_methods_by_file = files_to_spec.each_with_object({}) do |file, h|
    methods = git.diff_for_file(file)
                 .patch
                 .split("\n")
                 .grep(new_method_regex)
                 .reject { |line| line.match?(/def initialize/) }
                 .map { |l| [l[new_method_regex, 1], line_number_from_diff(file, l)] }
                 .to_h
    next if methods.empty?

    h[file] = methods
  end

  new_methods_by_file.each do |file, methods|
    spec_file = "spec/" + file.sub(".rb", "_spec.rb").sub("app/", "")
    next warn("No spec found for file #{file}.") unless File.exist?(spec_file)

    specs = IO.readlines(spec_file)
              .map { |line| line[spec_regex, 1] }
              .to_set
              .tap { |set| set.delete(nil) }
    methods.each do |method, line_no|
      next if specs.include?(method)

      warn("Missing spec for `#{method}`", file: file, line: line_no)
    end
  end
end


warn_zero_downtime
warn_no_spec_for_method
warn_rubocop
