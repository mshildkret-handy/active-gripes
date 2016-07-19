# ActiveGripes

```ruby
MyProblems::With::ActiveRecord
```

* What does `<<` actually do for a `has_many` relation
* Use of transactions with `find_or_create_by`
* Racing to `find_or_create` records!
* PSA: `save!` vs. `save`
* Migrations and Caching


**NB: All lessons learned the hard way.**

**NB2: The AccountingService uses AR 4.2.4**

<hr>

### Strange `<<` Behavior

* Lets say are creating a `foo` and `bar`.
* `foo` has many `bars` in the rails way.

```ruby
> foo.persisted? 
#=> false
> bar.persisted? 
#=> false
> foo.bars << bar # Does not hit the DB
#=> <ActiveRecord::Associations::CollectionProxy [<Bar id: nil quxx: nil>]>
> foo.save! # hits the DB! All records are persisted 
#=> true
``` 

<hr>

### Strange `<<` Behavior

* Now `foo` and `bar` need updating!
* What happens when we run the same code!

```ruby
> foo.persisted? 
#=> true
> bar.persisted? 
#=> true
> foo.bars << bar # HITS THE DB! Creates the association.
#=> <ActiveRecord::Associations::CollectionProxy [<Bar id: 1 quxx: nil>, <Bar id: 1 quxx: nil>]>
> foo.save! # The association was already created above.
#=> true
``` 

Be careful of this when using `has_many`.

<hr>

### `find_or_create_by` in `transaction`s


*  Again, `foo` has_many `bars`. 
*  We want to build `bars` in a transaction and add them to a `Foo`.

In the example below, if will create 5 new bars if 1 does not exist before transaction is created

```ruby
Foo.transaction do
  5.times do 
    bar_five = bar.find_or_create_by(quxx: 5) # created in transaction
    foo.bars << bar_five # adds 5 bar_fives to foo
  end
end
```
<hr>

### `find_or_create_by` in `transaction`s

* This behavior is logical because the objects returned by `find_or_create_by` are unpersisted during the transaction.
* `find_or_create_by` looks for a new `bar`, finds nothing, and returns a new object.
* It is important to understand that while in the transaction, `find_or_create_by` has no idea of your objects in memory.

<hr>

### `persisted?` & `reload` behave strange around rollbacks

*But first, a tale from the Accounting Service...*

* Double entry accounting means you are never creating just one ledger entry.
* Rows are inserted into the `ledger_entries` table in twos.

The code to do that looks something like this:

```ruby
  def self.persist_ledger_entries(le_one, le_two)
    validate_transaction(le_one, le_two)
    le_one.save!
    le_two.save!

    le_one.transaction_pair_id = le_two.id
    le_two.transaction_pair_id = le_one.id

    le_one.save!
    le_two.save!
  end
```
 
 <hr>
 
### `persisted?` & `reload` behave strange around rollbacks

* Often times transactions trigger ledger entries across many different accounts

We provide an interface for this in the method `create_transactions`:

```ruby
  # transactions is an array of ledger entry pairs
  # example: [[entry1, entry2], [entry1, entry2]]
  def self.create_transactions!(transactions)
    LedgerEntry.transaction do
      transactions.each do |transaction|
        self.persist_ledger_entries(*transaction)
      end
    end
  end

```

<hr>

### `persisted?` & `reload` behave strange around rollbacks

* The rollback properly cleans up the database, however our ruby objects are left in a partial state. 
* `save!` calls `create_or_update`, so it looks like when you call `save!` twice in a `transaction` block, `ActiveRecord` does not refresh the whole object.

So that can lead to places like this:

```ruby
foo = Foo.new
ActiveRecord::Base.transaction do
  foo.save!
  foo.save!
  raise ActiveRecord::Rollback
end

foo.persisted?
#=> true
foo.id 
#=> 1
foo.reload # Raises Error!
# ActiveRecord::RecordNotFound:\
# Couldn't find BusinessObject with 'id'=1
```

<hr>

### Race conditions with `find_or_create_by` and `validates_uniqueness_of`

* Sinatra1 and Sinatra2 both hit MySql one after another and try to create identical Foo objects.
* Foo has a `validates_uniqueness_of` constraint on `buzz_type`, which Sinatra2 is violating.
* calling `save!` on the Foo in Sinatra2 will raise a `ActiveRecord::RecordInvalid` error since the object in memory is now invalid.

<hr> 

### Race conditions with `find_or_create_by` and `validates_uniqueness_of`

* [To ActiveRecord's credit, this is a well documented issue. See the docs here:](http://www.rubydoc.info/gems/activerecord/4.2.4/ActiveRecord/Relation#find_or_create_by-instance_method):

> Please note *this method is not atomic*, it runs first a SELECT, and if there are no results an INSERT is attempted. If there are other threads or processes there is a race condition between both calls and it could be the case that you end up with two similar records.

<hr>

### When `save` attacks!

* Unclear behavior when `save` raises and when it returns false.
* `save` will return `false` if an ActiveRecord *validation* fails. This indicates that the record was not persisted. 
* `save` will **raise an `ActiveRecord::WrappedDatabaseException`** if the record validates, but something at the DB level (such as a unique index) causes the save to fail.

<hr>

### When `save` attacks!

The gripe here is it seems to violate the law of least suprise. The way people explain `save` vs `save!` is that one returns `false` and the other raises. In truth they both can raise an error, its just one will raise due to issues on the ruby object or DB, where the other will just raise on DB issues.

### Migrations and Caching
ActiveRecord caches table columns, and uses this cache to build INSERT statements. Even if the code is not touching that column, ActiveRecord will still attempt to set it to NULL when saving models. The cache will stay until a hard restart of rails occurs, or some equivalent. 

Therefore, assuming zero downtime during a migration deploy, there is a window where Rails believes that column is still there and will error out on any INSERT (or create in ruby) until Rails is restarted. With a rolling deploy process, this period of time in between is inescapable.

One way to ensure 100% uptime with no errors is to add some code to the model to force ActiveRecord to ignore the column. Let's say you have a table/model called users/User and a column you wish to remove called "useless". Assuming all references have already been deleted, add this code before doing the migration:

```ruby
class User
  def self.columns
    super.reject { |c| c.name == "useless" }
  end
end
```
While annoying to have to add this code, and then delete it after the deploy, it is probably the only way of ensuring 100% uptime with no errors. 
