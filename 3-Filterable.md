## Introduction
Lets say we want to return a list of users filtered by multiple parameters. When we navigate to:

`/users?name=er&last_name=&company_id=2&roles[]=1&roles[]=4&roles[]=7&industry=5`

`$request->all()` will return:

```php
[
    'name'       => 'er',
    'last_name'  => '',
    'company_id' => '2',
    'roles'      => ['1','4','7'],
    'industry'   => '5'
]
```

To filter by all those parameters we would need to do something like:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Http\Requests;
use App\User;

class UserController extends Controller
{

    public function index(Request $request)
    {
        $query = User::where('company_id', $request->input('company_id'));

        if ($request->has('last_name'))
        {
            $query->where('last_name', 'LIKE', '%' . $request->input('last_name') . '%');
        }

        if ($request->has('name'))
        {
            $query->where(function ($q) use ($request)
            {
                return $q->where('first_name', 'LIKE', $request->input('name') . '%')
                    ->orWhere('last_name', 'LIKE', '%' . $request->input('name') . '%');
            });
        }

        $query->whereHas('roles', function ($q) use ($request)
        {
            return $q->whereIn('id', $request->input('roles'));
        })
            ->whereHas('clients', function ($q) use ($request)
            {
                return $q->whereHas('industry_id', $request->input('industry'));
            });

        return $query->get();
    }

}
```

To filter that same input With Eloquent Filters:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Http\Requests;
use App\User;

class UserController extends Controller
{

    public function index(Request $request)
    {
        return User::filter($request->all())->get();
    }

}
```

## Configuration
### Install Through Composer
```
composer require islem-kms/filterable
```

There are a few ways to define the filter a model will use:

- [Use EloquentFilter's Default Settings](#default-settings)
- [Use A Custom Namespace For All Filters](#with-configuration-file-optional)
- [Define A Model's Default Filter](#define-the-default-model-filter)
- [Dynamically Select A Model's Filter](#dynamic-filters)


#### Default Settings
The default namespace for all filters is `App\ModelFilters\` and each Model expects the filter classname to follow the `{$ModelName}Filter` naming convention regardless of the namespace the model is in.  Here is an example of Models and their respective filters based on the default naming convention.

| Model                           | ModelFilter                          |
| ------------------------------- | ------------------------------------ |
| `App\User`                      | `App\ModelFilters\UserFilter`        |
| `App\FrontEnd\PrivatePost`      | `App\ModelFilters\PrivatePostFilter` |
| `App\FrontEnd\Public\GuestPost` | `App\ModelFilters\GuestPostFilter`   |


#### Define The Default Model Filter

Create a public method `modelFilter()` that returns `$this->provideFilter(Your\Model\Filter::class);` in your model.

```php
<?php

namespace App;

use IslemKms\EloquentFilter\Filterable;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    use Filterable;

    public function modelFilter()
    {
        return $this->provideFilter(App\ModelFilters\CustomFilters\CustomUserFilter::class);
    }

    //User Class
}
```
#### Dynamic Filters

You can define the filter dynamically by passing the filter to use as the second parameter of the `filter()` method.  Defining a filter dynamically will take precedent over any other filters defined for the model.

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Http\Requests;
use App\User;
use App\ModelFilters\Admin\UserFilter as AdminFilter;
use App\ModelFilters\User\UserFilter as BasicUserFilter;
use Auth;

class UserController extends Controller
{
    public function index(Request $request)
    {
        $userFilter = Auth::user()->isAdmin() ? AdminFilter::class : BasicUserFilter::class;

        return User::filter($request->all(), $userFilter)->get();
    }
}

```

## Usage

### Defining The Filter Logic
Define the filter logic based on the camel cased input key passed to the `filter()` method.

- Empty strings are ignored
- `setup()` will be called regardless of input
- `_id` is dropped from the end of the input to define the method so filtering `user_id` would use the `user()` method
- Input without a corresponding filter method are ignored
- The value of the key is injected into the method
- All values are accessible through the `$this->input()` method or a single value by key `$this->input($key)`
- All Eloquent Builder methods are accessible in `this` context in the model filter class.

To define methods for the following input:

```php
[
    'company_id'   => 5,
    'name'         => 'Tuck',
    'mobile_phone' => '888555'
]
```

You would use the following methods:

```php

use IslemKms\EloquentFilter\ModelFilter;

class UserFilter extends ModelFilter
{
    protected $blacklist = ['secretMethod'];
    
    // This will filter 'company_id' OR 'company'
    public function company($id)
    {
        return $this->where('company_id', $id);
    }

    public function name($name)
    {
        return $this->where(function($q) use ($name)
        {
            return $q->where('first_name', 'LIKE', "%$name%")
                ->orWhere('last_name', 'LIKE', "%$name%");
        });
    }

    public function mobilePhone($phone)
    {
        return $this->where('mobile_phone', 'LIKE', "$phone%");
    }

    public function setup()
    {
        $this->onlyShowDeletedForAdmins();
    }

    public function onlyShowDeletedForAdmins()
    {
        if(Auth::user()->isAdmin())
        {
            $this->withTrashed();
        }
    }
    
    public function secretMethod($secretParameter)
    {
        return $this->where('some_column', true);
    }
}
```

#### Additional Filter Methods

The `Filterable` trait also comes with the below query builder helper methods:

| EloquentFilter Method                      | QueryBuilder Equivalent                           |
| ------------------------------------------ | ------------------------------------------------- |
| `$this->whereLike($column, $string)`       | `$query->where($column, 'LIKE', '%'.$string.'%')` |
| `$this->whereBeginsWith($column, $string)` | `$query->where($column, 'LIKE', $string.'%')`     |
| `$this->whereEndsWith($column, $string)`   | `$query->where($column, 'LIKE', '%'.$string)`     |

Since these methods are part of the `Filterable` trait they are accessible from any model that implements the trait without the need to call in the Model's EloquentFilter.


### Applying The Filter To A Model

Implement the `IslemKms\EloquentFilter\Filterable` trait on any Eloquent model:

```php
<?php

namespace App;

use IslemKms\EloquentFilter\Filterable;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    use Filterable;

    //User Class
}
```

This gives you access to the `filter()` method that accepts an array of input:

```php
class UserController extends Controller
{
    public function index(Request $request)
    {
        return User::filter($request->all())->get();
    }
}
```
