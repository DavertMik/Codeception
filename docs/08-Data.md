# Working with Data

Tests should not affect each other. That's a rule. When tests interact with databases they may change data inside them. This leads to data inconsistency. Test may try to insert the record is already inserted, or retrieve deleted record. To avoid test fails, database should be brought to it's initial state. Codeception has different methods and approaches to get your data cleaned.

## Cleanup

This chapter summarizes all notices on clean ups from previous chapter and suggests you best strategies to choose data storage backends.

When we choose to clean up database, we should make this cleaning to be as fast as possible. Test should always run fast. Rebuilding database from scratch is not the best way, but might be the only one. In any way, you should use special test database for testing. Do not ever test on development or production database!

Codeception has a Db module, which takes the most tasks on database interaction. By default it is trying to repopulate database from dump and clean it up after each test. This module expects a database dump in SQL format. It's already prepared for configuration in codeception.yml

``` yaml
modules:
    config:
        Db:
            dsn: 'PDO DSN HERE'
            user: 'root'
            password:
            dump: tests/_data/your-dump-name.sql
```

After you enable this module in your test suite it will automatically populate database from dump and repopulate on each test run. This settings can be changed through _populate_ and _cleanup_ options, which may be set to disabled.

Db module is a rough tool. It acts for any kind of databases supported by PDO. It could be used for all tests If it wasn't that slow. Loading dump takes pretty much time, that can be saved by other technics. When your test and application share the same connection, as it may be in functional and unit test, best way to speed up everything - put all code in transactions, or use SQLite Memory. When database interactions are performed through different connections, as we do in acceptance tests, the best solution we can propose - use SQLite file database, replace it after each test, instead of rebuild.

## Separated connections

In acceptance test your test is interacting with application through a web server. There is no way to receive database connection from web server, and this means test and application will work with same database but on different connections. Provide in Db module the same credentials your application use and then you can access database for assertions (seeInDabatase actions) and perform automatic cleanups.

### Speedup with SQLite

To speed up testing we recommend you to try using SQLite in your application. Convert your database dump to SQLite format. Use one of [provided tools](http://www.sqlite.org/cvstrac/wiki?p=ConverterTools).

Keep in mind that columns in SQLite are case-sensitive. So it's important to set 'COLLATE NOCASE' for each text and varchar columns.

Store SQLite test database in 'tests/_data' directory. And point Db module to it too:

``` yaml
modules:
    config:
        Db:
            dsn: 'sqlite:tests/_data/application.db'
            user: 'root'
            password:
            dump: tests/_data/sqlite_dump.sql
```

Before test suite is started SQLite database is created and copied. After each test copy replaces database and reconnection is done. 

## Shared connections

When application or it's parts are run within Codeception process, you can use your application connection in your tests. 
If you get an access to connection, all database operations can be put into one global transaction and rolled back at the end. That will dramatically improve performance. Nothing will be written to database at the end, thus no database repopulation is actually needed.

### ORM modules

If your application is using ORMs like Doctrine or Doctrine2, connect the respected modules to suite. By default they will cover everything in transaction. If your use several database connections, or there are transactions not tracked by ORM, that module can be useless for you.

ORM module can be connected with Db module, but by default both will perform cleanup. Thus you should explicitly set which module is used:

In tests/functional.suite.yml:

``` yaml
modules:
	enabled: [Db, Doctrine1, TestHelper]
	config:
		Db:
			cleanup: false
```

Still, Db module will perform database population from dump before each test. Use 'populate: false' to disable it.

### Dbh module

If you use PostgreSQL, or any other database which supports nested transactions, you can use Dbh module. It takes a PDO instance from your application, starts a transaction in the beginning of tests, and rolls it back at the end.
A PDO connection can be set in bootstrap file. This module also overrides _seeInDatabase_ and _dontSeeInDatabase_ actions of Db module.
To use Db module for population and Dbh for cleanups, use this config:

``` yaml
modules:
	enabled: [Db, Dbh, TestHelper]
	config:
		Db:
			cleanup: false

```

Please, note that Dbh module goes after Db. That allows Dbh module to override actions.

## Fixtures

Fixtures is a sample data we can use in tests. This data can be either generated or taken from sample database. Fixtures should be set in bootstrap file. Depending on suite a fixture definitions may change. 

#### Fixtures for Acceptance and Functional Tests

For acceptance and functional tests fixtures can be simply defined and used.

Fixtures definition in __bootstrap.php_
``` php
<?php
// let's take user from sample database. 
// we can populate it with Db module
$davert = Doctrine::getTable('User')->findOneBy('name', 'davert');

?>
```

Fixture usage sample accetpance or functional test.

``` php
<?php
$I = new TestGuy($scenario);
$I->amLoggedAs($davert);
$I->see('Welcome, Davert');
?>
```

All variables from bootstrap file are passed to Cept files of testing suite. 
You can use [Faker](https://github.com/fzaninotto/Faker) library to create tested data within bootstrap file.

#### Fixtures for Unit Tests.

Passing fixtures to Cest files are bit harder, as you can't pass variable into class. Thus, a special registry class Fixtures is used.

Fixture definition in __bootstrap.php_ for unit test suite.

``` php
<?php
use \Codeception\Util\Fixtures as Fixtures;
Fixtures::add('davert', Doctrine::getTable('User')->findOneBy('name', 'davert'));
?>
```

In sample unit test:

``` php
<?php
$I->execute(function () {
    $user = Fixtures::get('davert');
    return $user->getName();        
});
$I->seeResultEquals('davert');
?>
```

Fixtures class stores and retrieves any variable by key. You can extend it by adding namespaces.

In __bootstrap.yml_

``` php
<?php
use \Codeception\Util\Fixtures as Fixtures;

class MyFixtures extends Fixtures {
    public static function add($namespace, $key, $value) {
        parent::add("$namespace.$key", $value);
    }    
    public static function get($namespace, $key) {
        return parent::get("$namespace.$key");
    }

    public static function getUser($key)
    {
        return parent::get("user.$key");
    }
};

MyFixtures::add('user', 'davert', Doctrine::getTable('User')->findOneBy('name', 'davert'));

?>
```


In test:

``` php
<?php
$I->execute(function () {
    $user = Fixtures::get('user','davert');
    // or
    $user = Fixtures::getUser('davert');
    return $user->getName();        
});
$I->seeResultEquals('davert');
?>
```
?>
```

Using fixtures simplifies your testing code. Name your fixtures wisely and your tests would gain in readability.

## Conclusion

Codeception is not leaving developer alone in dealing with data. Tools for database population and cleanups are bundled within Db module.
To manipulate sample data in test use fixtures that can be defined within bootstrap file.