require 'active_support/core_ext/string'

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
  rubocop.lint(git.modified_files + git.added_files)
end

warn_zero_downtime
warn_rubocop
