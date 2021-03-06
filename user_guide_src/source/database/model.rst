#########################
Using CodeIgniter's Model
#########################

.. contents::
    :local:
    :depth: 2

Manual Model Creation
=====================

You do not need to extend any special class to create a model for your application. All you need is to get an
instance of the database connection and you're good to go.

::

	use \CodeIgniter\Database\ConnectionInterface;

	class UserModel
	{
		protected $db;

		public function __construct(ConnectionInterface &$db)
		{
			$this->db =& $db;
		}
	}

CodeIgniter's Model
===================

CodeIgniter does provide a model class that provides a few nice features, including:

- automatic database connection
- basic CRUD methods
- in-model validation
- automatic pagination
- and more

This class provides a solid base from which to build your own models, allowing you to
rapidly build out your application's model layer.

Creating Your Model
===================

To take advantage of CodeIgniter's model, you would simply create a new model class
that extends ``CodeIgniter\Model``::

	class UserModel extends \CodeIgniter\Model
	{

	}

This empty class provides convenient access to the database connection, the Query Builder,
and a number of additional convenience methods.

Connecting to the Database
--------------------------

When the class is first instantiated, if no database connection instance is passed to constructor,
it will automatically connect to the default database group, as set in the configuration. You can
modify which group is used on a per-model basis by adding the DBGroup property to your class.
This ensures that within the model any references to ``$this->db`` are made through the appropriate
connection.
::

	class UserModel extends \CodeIgniter\Model
	{
		protected $DBGroup = 'group_name';
	}

You would replace "group_name" with the name of a defined database group from the database
configuration file.

Configuring Your Model
----------------------

The model class has a few configuration options that can be set to allow the class' methods
to work seamlessly for you. The first two are used by all of the CRUD methods to determine
what table to use and how we can find the required records::

	class UserModel extends \CodeIgniter\Model
	{
		protected $table      = 'users';
		protected $primaryKey = 'id';

		protected $returnType = 'array';
		protected $useSoftDeletes = true;

		protected $allowedFields = ['name', 'email'];

		protected $useTimestamps = false;

		protected $validationRules    = [];
		protected $validationMessages = [];
		protected $skipValidation     = false;
	}

**$table**

Specifies the database table that this model primarily works with. This only applies to the
built-in CRUD methods. You are not restricted to using only this table in your own
queries.

**$primaryKey**

This is the name of the column that uniquely identifies the records in this table. This
does not necessarilly have to match the primary key that is specified in the database, but
is used with methods like ``find()`` to know what column to match the specified value to.

**$returnType**

The Model's CRUD methods will take a step of work away from you and automatically return
the resulting data, instead of the Result object. This setting allows you to define
the type of data that is returned. Valid values are 'array', 'object', or the fully
qualified name of a class that can be used with the Result object's getCustomResultObject()
method.

**$useSoftDeletes**

If true, then any delete* method calls will simply set a flag in the database, instead of
actually deleting the row. This can preserve data when it might be referenced elsewhere, or
can maintain a "recycle bin" of objects that can be restored, or even simply preserve it as
part of a security trail. If true, the find* methods will only return non-deleted rows, unless
the withDeleted() method is called prior to calling the find* method.

This requires an INT or TINYINT field to be present in the table for storing state.The default field name is  ``deleted`` however this name can be configured to any name of your choice by using $deletedField property.

**$allowedFields**

This array should be updated with the field names that can be set during save, insert, or
update methods. Any field names other than these will be discarded. This helps to protect
against just taking input from a form and throwing it all at the model, resulting in
potential mass assignment vulnerabilities.

**$useTimestamps**

This boolean value determines whether the current date is automatically added to all inserts
and updates. If true, will set the current time in the format specified by $dateFormat. This
requires that the table have columns named 'created_at' and 'updated_at' in the appropriate
data type.

**$dateFormat**

This value works with $useTimestamps to ensure that the correct type of date value gets
inserted into the database. By default, this creates DATETIME values, but valid options
are: datetime, date, or int (a PHP timestamp).

**$validationRules**

Contains either an array of validation rules as described in :ref:`validation-array`
or a string containing the name of a validation group, as described in the same section.
Described in more detail below.

**$validationMessages**

Contains an array of custom error messages that should be used during validation, as
described in :ref:`validation-custom-errors`. Described in more detail below.

**$skipValidation**

Whether validation should be skipped during all ``inserts`` and ``updates``. The default
value is false, meaning that data will always attempt to be validated. This is
primarily used by the ``skipValidation()`` method, but may be changed to ``true`` so
this model will never validate.

**$beforeInsert**
**$afterInsert**
**$beforeUpdate**
**$afterUpdate**
**afterFind**
**afterDelete**

These arrays allow you to specify callback methods that will be ran on the data at the
time specified in the property name.

Working With Data
=================

Finding Data
------------

Several functions are provided for doing basic CRUD work on your tables, including find(),
insert(), update(), delete() and more.

**find()**

Returns a single row where the primary key matches the value passed in as the first parameter::

	$user = $userModel->find($user_id);

The value is returned in the format specified in $returnType.

You can specify more than one row to return by passing an array of primaryKey values instead
of just one::

	$users = $userModel->find([1,2,3]);

**findWhere()**

Allows you to specify one or more criteria that must be matched against the data. Returns
all rows that match::

	// Use simple where
	$users = $userModel->findWhere('role_id >', '10');

	// Use array of where values
	$users = $userModel->findWhere([
		'status'  => 'active',
		'deleted' => 0
	]);

**findAll()**

Returns all results::

	$users = $userModel->findAll();

This query may be modified by interjecting Query Builder commands as needed prior to calling this method::

	$users = $userModel->where('active', 1)
	                   ->findAll();

You can pass in a limit and offset values as the first and second
parameters, respectively::

	$users = $userModel->findAll($limit, $offset);

**first()**

Returns the first row in the result set. This is best used in combination with the query builder.
::

	$user = $userModel->where('deleted', 0)
	                  ->first();

**withDeleted()**

If $useSoftDeletes is true, then the find* methods will not return any rows where 'deleted = 1'. To
temporarily override this, you can use the withDeleted() method prior to calling the find* method.
::

	// Only gets non-deleted rows (deleted = 0)
	$activeUsers = $userModel->findAll();

	// Gets all rows
	$allUsers = $userModel->withDeleted()
	                      ->findAll();

**onlyDeleted()**

Whereas withDeleted() will return both deleted and not-deleted rows, this method modifies
the next find* methods to return only soft deleted rows::

	$deletedUsers = $userModel->onlyDeleted()
	                          ->findAll();

Saving Data
-----------

**insert()**

An associative array of data is passed into this method as the only parameter to create a new
row of data in the database. The array's keys must match the name of the columns in $table, while
the array's values are the values to save for that key::

	$data = [
		'username' => 'darth',
		'email'    => 'd.vader@theempire.com'
	];

	$userModel->insert($data);

**update()**

Updates an existing record in the database. The first parameter is the $primaryKey of the record to update.
An associative array of data is passed into this method as the second parameter. The array's keys must match the name
of the columns in $table, while the array's values are the values to save for that key::

	$data = [
		'username' => 'darth',
		'email'    => 'd.vader@theempire.com'
	];

	$userModel->update($id, $data);

**save()**

This is a wrapper around the insert() and update() methods that handles inserting or updating the record
automatically, based on whether it finds an array key matching the $primaryKey value::

	// Defined as a model property
	$primaryKey = 'id';

	// Does an insert()
	$data = [
		'username' => 'darth',
		'email'    => 'd.vader@theempire.com'
	];

	$userModel->save($data);

	// Performs an update, since the primary key, 'id', is found.
	$data = [
		'id'       => 3,
		'username' => 'darth',
		'email'    => 'd.vader@theempire.com'
	];
	$userModel->save($data);

The save method also can make working with custom class result objects much simpler by recognizing a non-simple
object and grabbing its public and protected values into an array, which is then passed to the appropriate
insert or update method. This allows you to work with Entity classes in a very clean way. Entity classes are
simple classes that represent a single instance of an object type, like a user, a blog post, job, etc. This
class is responsible for maintaining the business logic surrounding the object itself, like formatting
elements in a certain way, etc. They shouldn't have any idea about how they are saved to the database. At their
simplest, they might look like this::

	namespace App\Entities;

	class Job
	{
		protected $id;
		protected $name;
		protected $description;

		public function __get($key)
		{
			if (property_exists($this, $key))
			{
				return $this->$key;
			}
		}

		public function __set($key, $value)
		{
			if (property_exists($this, $key))
			{
				$this->$key = $value;
			}
		}
	}

A very simple model to work with this might look like::

	class JobModel extends \CodeIgniter\Model
	{
		protected $table = 'jobs';
		protected $returnType = '\App\Entities\Job';
		protected $allowedFields = [
			'name', 'description'
		];
	}

This model works with data from the ``jobs`` table, and returns all results as an instance of ``App\Entities\Job``.
When you need to persist that record to the database, you will need to either write custom methods, or use the
model's ``save()`` method to inspect the class, grab any public and private properties, and save them to the database::

	// Retrieve a Job instance
	$job = $model->find(15);

	// Make some changes
	$job->name = "Foobar";

	// Save the changes
	$model->save($job);

.. note:: If you find yourself working with Entities a lot, CodeIgniter provides a built-in :doc:`Entity class </database/entities>`
	that provides several handy features that make developing Entities simpler.

Deleting Data
-------------

**delete()**

Takes a primary key value as the first parameter and deletes the matching record from the model's table::

	$userModel->delete(12);

If the model's $useSoftDeletes value is true, this will update the row to set 'deleted = 1'. You can force
a permanent delete by setting the second parameter as true.

**deleteWhere()**

Deletes multiple records from the model's table based on the criteria pass into the first two parameters.
::

	// Simple where
	$userMdoel->deleteWhere('status', 'inactive');

	// Complex where
	$userModel->deleteWhere([
		'status'      => 'inactive',
		'warn_lvl >=' => 50
	]);

If the model's $useSoftDeletes value is true, this will update the rows to set 'deleted = 1'. You can force
a permanent delete by setting the third parameter as true.

**purgeDeleted()**

Cleans out the database table by permanently removing all rows that have 'deleted = 1'. ::

	$userModel->purgeDeleted();

Validating Data
---------------

For many people, validating data in the model is the preferred way to ensure the data is kept to a single
standard, without duplicating code. The Model class provides a way to automatically have all data validated
prior to saving to the database with the ``insert()``, ``update()``, or ``save()`` methods.

The first step is to fill out the ``$validationRules`` class property with the fields and rules that should
be applied. If you have custom error message that you want to use, place them in the ``$validationMessages`` array::

	class UserModel extends Model
	{
		protected $validationRules    = [
			'username'     => 'required|alpha_numeric_space|min_length[3]',
			'email'        => 'required|valid_email|is_unique[users.email]',
			'password'     => 'required|min_length[8]',
			'pass_confirm' => 'required_with[password]|matches[password]'
		];

		protected $validationMessages = [
			'email'        => [
				'is_unique' => 'Sorry. That email has already been taken. Please choose another.'
			]
		];
	}

Now, whenever you call the ``insert()``, ``update()``, or ``save()`` methods, the data will be validated. If it fails,
the model will return boolean **false**. You can use the ``errors()`` method to retrieve the validation errors::

	if ($model->save($data) === false)
	{
		return view('updateUser', ['errors' => $model->errors()];
	}

This returns an array with the field names and their associated errors that can be used to either show all of the
errors at the top of the form, or to display them individually::

	<?php if (! empty($errors)) : ?>
		<div class="alert alert-danger">
		<?php foreach ($errors as $field => $error) : ?>
			<p><?= $error ?></p>
		<?php endforeach ?>
		</div>
	<?php endif ?>

If you'd rather organize your rules and error messages within the Validation configuration file, you can do that
and simply set ``$validationRules`` to the name of the validation rule group you created::

	class UserModel extends Model
	{
		protected $validationRules = 'users';
	}

Protecting Fields
-----------------

To help protect against Mass Assignment Attacks, the Model class **requires** that you list all of the field names
that can be changed during inserts and updates in the ``$allowedFields`` class property. Any data provided
in addition to these will be removed prior to hitting the database. This is great for ensuring that timestamps,
or primary keys do not get changed.
::

	protected $allowedFields = ['name', 'email', 'address'];

Occasionally, you will find times where you need to be able to change these elements. This is often during
testing, migrations, or seeds. In these cases, you can turn the protection on or off::

	$model->protect(false)
	      ->insert($data)
	      ->protect(true);

Working With Query Builder
--------------------------

You can get access to a shared instance of the Query Builder for that model's database connection any time you
need it::

	$builder = $userModel->builder();

This builder is already setup with the model's $table.

You can also use Query Builder methods and the Model's CRUD methods in the same chained call, allowing for
very elegant use::

	$users = $userModel->where('status', 'active')
			   ->orderBy('last_login', 'asc')
			   ->findAll();

.. note:: You can also access the model's database connection seamlessly::

			$user_name = $userModel->escape($name);

Runtime Return Type Changes
----------------------------

You can specify the format that data should be returned as when using the find*() methods as the class property,
$returnType. There may be times that you would like the data back in a different format, though. The Model
provides methods that allow you to do just that.

.. note:: These methods only change the return type for the next find*() method call. After that,
			it is reset to its default value.

**asArray()**

Returns data from the next find*() method as associative arrays::

	$users = $userModel->asArray()->findWhere('status', 'active');

**asObject()**

Returns data from the next find*() method as standard objects or custom class intances::

	// Return as standard objects
	$users = $userModel->asObject()->findWhere('status', 'active');

	// Return as custom class instances
	$users = $userModel->asObject('User')->findWhere('status', 'active');

Processing Large Amounts of Data
--------------------------------

Sometimes, you need to process large amounts of data and would run the risk of running out of memory.
To make this simpler, you may use the chunk() method to get smaller chunks of data that you can then
do your work on. The first parameter is the number of rows to retrieve in a single chunk. The second
parameter is a Closure that will be called for each row of data.

This is best used during cronjobs, data exports, or other large tasks.
::

	$userModel->chunk(100, function ($data)
	{
		// do something.
		// $data is a single row of data.
	});

Model Events
============

There are several points within the model's execution that you can specify multiple callback methods to run.
These methods can be used to normalize data, hash passwords, save related entities, and much more. The following
points in the model's execution can be affected, each through a class property: **$beforeInsert**, **$afterInsert**,
**$beforeUpdate**, **afterUpdate**, **afterFind**, and **afterDelete**.

Defining Callbacks
------------------

You specify the callbacks by first creating a new class method in your model to use. This class will always
receive a $data array as its only parameter. The exact contents of the $data array will vary between events, but
will always contain a key named **data** that contains the primary data passed to original method. In the case
of the insert* or update* methods, that will be the key/value pairs that are being inserted into the database. The
main array will also contain the other values passed to the method, and be detailed later. The callback method
must return the original $data array so other callbacks have the full information.

::

	protected function hashPassword(array $data)
	{
		if (! isset($data['data']['password']) return $data;

		$data['data']['password_hash'] = password_hash($data['data']['password'], PASSWORD_DEFAULT);
		unse($data['data']['password'];

		return $data;
	}

Specifying Callbacks To Run
---------------------------

You specify when to run the callbacks by adding the method name to the appropriate class property (beforeInsert, afterUpdate,
etc). Multiple callbacks can be added to a single event and they will be processed one after the other. You can
use the same callback in multiple events::

	protected $beforeInsert = ['hashPassword'];
	protected $beforeUpdate = ['hashPassword'];

Event Parameters
----------------

Since the exact data passed to each callback varies a bit, here are the details on what is in the $data parameter
passed to each event:

================ =========================================================================================================
Event            $data contents
================ =========================================================================================================
beforeInsert	  **data** = the key/value pairs that are being inserted. If an object or Entity class is passed to the insert
				  method, it is first converted to an array.
afterInsert		  **data** = the original key/value pairs being inserted. **result** = the results of the insert() method
				  used through the Query Builder.
beforeUpdate	  **id** = the primary key of the row being updated. **data** = the key/value pairs that are being
				  inserted. If an object or Entity class is passed to the insert method, it is first converted to an array.
afterUpdate		  **id** = the primary key of the row being updated. **data** = the original key/value pairs being updated.
				  **result** = the results of the update() method used through the Query Builder.
afterFind		  Varies by find* method. See the following:
- find()		  **id** = the primary key of the row being searched for. **data** = The resulting row of data, or null if
				  no result found.
- findWhere()	  **data** = the resulting rows of data, or null if no result found.
- findAll()		  **data** = the resulting rows of data, or null if no result found. **limit** = the number of rows to find.
				  **offset** = the number of rows to skip during the search.
- first()		  **data** = the resulting row found during the search, or null if none found.
afterDelete		  Varies by delete* method. See the following:
- delete()		  **id** = primary key of row being deleted. **purge** boolean whether soft-delete rows should be
				  hard deleted. **result** = the result of the delete() call on the Query Builder. **data** = unused.
- deleteWhere()	  **key**/**value** = the key/value pair used to search for rows to delete. **purge** boolean whether
				  soft-delete rows should be hard deleted. **result** = the result of the delete() call on the Query
				  Builder. **data** = unused.
================ =========================================================================================================
