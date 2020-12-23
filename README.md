First off, this is a Rails 6.1 feature.

create rails app

Then, to point to the master branch, change the rails line to the below:

gem 'rails', github: "rails/rails", branch: "master"

Rebundle

Let's make a bunch of databases

Update your database.yml like below:

development:
  primary:
    <<: *default
    database: my_primary_database
  primary_replica:
    database: my_primary_database
    replica: true
  primary_shard_one:
    <<: *default
    database: my_primary_shard_one
  primary_shard_one_replica:
    <<: *default
    database: my_primary_shard_one
    replica: true
   
In this setup, we have a primary and primary_shard, and each has a replica.

Then run:

rails db:create; rails db:migrate

Let's generate a model:

rails g model Person name:string
rails db:migrate

#=>
== 20200303155421 CreatePeople: migrating ====================================
-- create_table(:animals)
   -> 0.0026s
== 20200303155421 CreatePeople: migrated (0.0028s) ===========================

== 20200303155421 CreatePeople: migrating ====================================
-- create_table(:animals)
   -> 0.0040s
== 20200303155421 CreatePeople: migrated (0.0041s) ===========================

And either in a seeds file or our console, run:
Person.create!(
  name: 'Frieda'
)

Person.create!(
  name: 'Bill'
)

Person.create!(
  name: 'Penelope'
)

Now, let's update our application_record.rb like so:

class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true

  connects_to shards: {
    default: { writing: :primary, reading: :primary_replica },
    shard_one: { writing: :primary_shard_one, reading: :primary_shard_one_replica }
  }
end

We will now reload our console and see what we can do, using ActiveRecord::Base.connected_to:

ActiveRecord::Base.connected_to(role: :reading, shard: :shard_one) do
  Person.first
end
#=> nil


# Now let's write to :shard_one,
ActiveRecord::Base.connected_to(role: :writing, shard: :shard_one) do
   Person.create!(
     name: 'John'
     )
    Person.create!(
      name: 'Georgina'
    )
 end

#=> #<Person id: 2, name: "Georgina", created_at: "2020-03-03 16:10:14", updated_at: "2020-03-03 16:10:14">

# And we'll read from :shard_one's replica:
ActiveRecord::Base.connected_to(role: :reading, shard: :shard_one) do
  Person.count
end
#=> 2

# And finally confirm that shard one's data has not been written to primary:

ActiveRecord::Base.connected_to(role: :reading, shard: :default) do
  Person.count
end
#=> 3

So this is database sharding
