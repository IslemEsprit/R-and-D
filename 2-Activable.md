## Prepare Model

To enable activation for a model, use the `IslemKms\Activable\Activable` trait on the model and add the `status`, `activated_by` and `activated_at` columns to your model's table.

```php
use IslemKms\Activable\Activable;
class Post extends Model
{
    use Activable;
    ...
}
```

Create a migration to add the new columns. [(You can use custom names for the activation columns)](#configuration)

Example Migration:

```php
class AddActivationColumnsToPostsTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::table('posts', function (Blueprint $table) {
            $table->smallInteger('status')->default(0);
            $table->dateTime('activated_at')->nullable();
            //If you want to track who activated the Model add 'activated_by' too.
            //$table->integer('activated_by')->nullable()->unsigned();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::table('posts', function(Blueprint $table)
        {
            $table->dropColumn('status');
            $table->dropColumn('activated_at');
            //$table->dropColumn('activated_by');
        });
    }
}
```

**You are ready to go!**

## Usage

> **Note:** In next examples I will use Post model to demonstrate how the query builder works. You can Moderate any Eloquent Model, even User.

### Moderate Models

You can moderate a model Instance:

```php
$post->markActive();

$post->markInactive();

$post->markPostponed();

$post->markPending();
```

or by referencing it's id

```php
Post::activate($post->id);

Post::deactivate($post->id);

Post::postpone($post->id);
```

or by making a query.

```php
Post::where('title', 'Horse')->activate();

Post::where('title', 'Horse')->deactivate();

Post::where('title', 'Horse')->postpone();
```

### Query Models

By default only Active models will be returned on queries. To change this behavior check the [configuration](#configuration).

##### To query the Active Posts, run your queries as always.

```php
//it will return all Active Posts (strict mode)
Post::all();

// when not in strict mode
Post::active()->get();

//it will return Active Posts where title is Horse
Post::where('title', 'Horse')->get();

```

##### Query pending or inactive models.

```php
//it will return all Pending Posts
Post::pending()->get();

//it will return all Inactive Posts
Post::inactive()->get();

//it will return all Postponed Posts
Post::postponed()->get();

//it will return Active and Pending Posts
Post::withPending()->get();

//it will return Active and Inactive Posts
Post::withInactive()->get();

//it will return Active and Postponed Posts
Post::withPostponed()->get();
```

##### Query ALL models

```php
//it will return all Posts
Post::withAnyStatus()->get();

//it will return all Posts where title is Horse
Post::withAnyStatus()->where('title', 'Horse')->get();
```

### Model Status

To check the status of a model there are 3 helper methods which return a boolean value.

```php
//check if a model is pending
$post->isPending();

//check if a model is active
$post->isActive();

//check if a model is inactive
$post->isInactive();

//check if a model is inactive
$post->isPostponed();
```

## Strict Activation

Strict Activation means that only Active resource will be queried. To query Pending resources along with Active you have to disable Strict Activation. See how you can do this in the [configuration](#configuration).

## Configuration

### Global Configuration

To configuration Activation package globally you have to edit `config/activable.php`.
Inside `activable.php` you can configure the following:

1. `status_column` represents the default column 'status' in the database.
2. `activated_at_column` represents the default column 'activated_at' in the database.
3. `activated_by_column` represents the default column 'activated_by' in the database.
4. `strict` represents [Strict Activation](#strict-activation).

### Model Configuration

Inside your Model you can define some variables to overwrite **Global Settings**.

To overwrite `status` column define:

```php
const ACTIVATION_STATUS = 'activation_status';
```

To overwrite `activated_at` column define:

```php
const ACTIVATED_AT = 'act_at';
```

To overwrite `activated_by` column define:

```php
const ACTIVATED_BY = 'act_by';
```

To enable or disable [Strict Activation](#strict-activation):

```php
public static $strictActivation = true;
```
