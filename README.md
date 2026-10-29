# Authentication and Authorisation

The starting point for this practical is the [**Intro to Laravel**](https://github.com/CHT2520-web-prog/intro-to-laravel) practical we did previously. You'll need to complete this first before attempting the following practical work.

## Authentication

### The Database

When we create a new Laravel project a _users_ table migration is created for us. Look under _databases/migrations_ to find this. The first thing we need to do is populate this table with some example users.

#### Seeding the Users Table

- Using Artisan create a new seeder for the _users_ table.

```
php artisan make:seeder UserSeeder
```

- Add some example users e.g.

```php
namespace Database\Seeders;

use Illuminate\Support\Facades\DB;
use Illuminate\Database\Console\Seeds\WithoutModelEvents;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\Hash;

class UserSeeder extends Seeder
{
    /**
     * Run the database seeds.
     */
    public function run(): void
    {
        DB::table('users')->insert(['name' => 'Kate Hutton', 'email' => 'k.l.hutton@hud.ac.uk', 'password' => Hash::make('password')]);
        DB::table('users')->insert(['name' => 'Yousef Miandad', 'email' => 'y.miandad@hud.ac.uk', 'password' => Hash::make('letmein')]);
        DB::table('users')->insert(['name' => 'Sunil Laxman', 'email' => 's.laxman@hud.ac.uk', 'password' => Hash::make('password2')]);
    }
}
```

- Most of this should be self-explanatory. The only new thing here is the use of the `Hash` class to generate a hash for each user's password.
- Add the _UserSeeder_ to the list of seeders in _DatabaseSeeder.php_. You may have some additional seeders if you completed the Eloquent practical work. 

```php
    public function run(): void
    {
        $this->call([
            FilmSeeder::class,
            UserSeeder::class
        ]);
    }
```

- Ask Artisan to re-generate the database tables.

```
php artisan migrate:fresh --seed
```

- In Adminer check you have some rows in the _users_ table and notice that the passwords have been hashed.

### Implementing Authentication

#### Creating a Log In View

- Create a new folder in the _resources/views_ folder, name it 'auth'.
- Create a new file in the _auth_ folder called _login.blade.php_.
- Add the following code in this file

```html
<x-layout title="Sign In">
  <h1>Sign In</h1>
  <form method="POST" action="/login">
    @csrf
    <div>
      <label for="email">Email:</label>
      <input type="text" id="email" name="email" />
    </div>
    <div>
      <label for="password">Password:</label>
      <input type="password" id="password" name="password" />
    </div>
    <div>
      <button type="submit">Sign in</button>
    </div>
  </form>
</x-layout>
```

#### Creating a Controller and Route

- Using Artisan create a new controller to handle authentication

```
php artisan make:controller AuthController
```

- Add an `index()` method to the controller

```php
public function index()
{
    return view('auth.login');
}
```

- In the _routes/web.php_ file add a route for the login form:

```php
Route::get('/login', [AuthController::class, 'index'])->name('login');
```
- We give the route a name of 'login'. By naming the route, Laravel will automatically redirect the user to this route if they are unauthenticated (not logged in). 
- You will also need to make sure you have imported the `AuthController` (look at the top of the file to see how we did this for the `FilmController`)

- Test this works by accessing the login route in a browser. You should see the login form.

#### Testing the Log In Details

- Look in _login.blade.php_, take note of the _action_ and _method_.
- In _web.php_ add a route to handle the log in form submission.

```php
Route::post('/login', [AuthController::class, 'login']);
```

- In `AuthController` import the `Auth` facade at the top of the file.

```php
use Illuminate\Support\Facades\Auth;
```

- Then add a `login()` method to handle the _login_ route

```php
public function login(Request $request)
{
    $userDetails = [
        "email" => $request->email,
        "password" => $request->password
    ];

    if (Auth::attempt($userDetails)) {
        $request->session()->regenerate();
        return redirect('/films');
    }
    return back();
}
```

- The first part of this method simply gets the user details from the form and puts them into an array.
- The [`Auth` facade](https://laravel.com/framework/docs/authentication#authenticating-users) performs authentication for us. The `attempt()` method will check these details against the _users_ table in the database. If we get a match we want to start a new session `$request->session()->regenerate();` and redirect the user to the homepage of the app.
- If the login attempt is unseccessfull we 'fall through' to the final line which sends the user back to the log in form.
- Test this works, look in the _UserSeeder.php_ file for usernames and passwords to test with.

#### Giving the User Some Feedback

- Even though this works, it isn't clear to the user that they have successfully signed in. We'll add some feedback in the layout file. 
- Open _resources/views/components/layout.blade.php_
- Add the following code immediately after the opening `<body>` tag and before the `<nav>`

```php
@auth
    <div>
        Logged in as {{Auth::user()->name}}
        <form method='POST' action='/logout'>
            @csrf
            <button type='submit'>Logout</button>
        </form>
    </div>
@endauth
```

The `@auth` blade directive is used to markout a block of code that is only for authenticated (logged in) users.

- Login into the website and you should see that the username is displayed at the top of the page.

#### Implementing Logout

- The form in this block of code submits to a URI of `/logout`.

- Add a route for this URI

```php
Route::post('/logout', [AuthController::class, 'logout']);
```

- Add a `logout()` method to the `AuthController`.

```php
public function logout(Request $request)
{
    Auth::logout();
    $request->session()->invalidate();
    $request->session()->regenerateToken();
    return redirect('/login');
}
```

- This simply logs the user out using the `Auth` facade, cleans up the session data and redirects them to the log in page.
- Test this works.

#### Providing a Sign In Link

- Back in _layout.blade.php_ modify the navigation options to include a sign-in option that will only be displayed for users that haven't logged in.
  - The [`@guest`](https://laravel.com/framework/docs/blade#authentication-directives) directive marks out code that will only be displayed for users that aren't authenticated.

```html
<nav>
  <ul>
    <li><a href="/films">Home</a></li>
    <li><a href="/films/create">Add new film</a></li>
    <li><a href="/films/about">About</a></li>
    @guest
    <li><a href="/login">Sign in</a></li>
    @endguest
  </ul>
</nav>
```

- Again test this works

### Protecting Routes

We want to prevent guest users from accessing certain pages in our site.

- First, in _index.blade.php_, we'll display different content depending on whether the user is logged in.
  - Edit this file to conditionally display the list of films. For example:

```html
<x-layout title="List the films">
      @auth
        <h1>Here's a list of films</h1>
        @foreach ($films as $film)
        <p>
            <a href="/films/{{$film->id}}">
                {{$film->title}}
            </a>
        </p>
        @endforeach
        {{ $films->links() }}
    @endauth

    @guest
    <p>
        You need to be logged in to view the content of this website.
    </p> 
    @endguest
</x-layout>
```

- This should make sense, it simply uses the `@auth` and `@guest` directives to display different content for different users.
- Check this works.

Even though we have hidden the navigation options it is still possible to access routes directly by entering a URI in the browser.

Next we will stop guest users from accessing certain routes. Open _routes/web.php_ and add [route middleware](https://laravel.com/framework/docs/authentication#protecting-routes) to prevent guest users from accessing the create, show and edit pages. You may have additional routes e.g. for certificates. You should protect these as well.

```php
use App\Http\Controllers\FilmController;
use App\Http\Controllers\AuthController;
use Illuminate\Support\Facades\Route;

Route::get('/login', [AuthController::class, 'index']);
Route::get('/login', [AuthController::class, 'index'])->name('login');
Route::post('/logout', [AuthController::class, 'logout']);

Route::get('/', [FilmController::class, 'index']);
Route::get('/films', [FilmController::class, 'index']);
Route::get('/films/create', [FilmController::class, 'create'])->middleware('auth');
Route::post('/films', [FilmController::class, 'store'])->middleware('auth');
Route::get('/films/{id}', [FilmController::class, 'show'])->middleware('auth');
Route::get('/films/{id}/edit', [FilmController::class, 'edit'])->middleware('auth');
Route::patch('/films/{id}', [FilmController::class, 'update'])->middleware('auth');
Route::delete('/films/{id}', [FilmController::class, 'destroy'])->middleware('auth');


Route::get('/about', function () {
    return view('pages.about');;
});
```

- Make sure you are logged out and try and access the _create_ route, you should find you get redirected to the login page.

## Authorisation Using Gates

Often we want to provide different levels of access for different types of users. To keep things simple we'll have two types of user, a regular user and an admin user.

### Setting up the database

We'll store a _role_id_ for each user in the _users_ database table.

- From the _database/migrations_ folder, open the migration for the _users_ table.
- Edit this file to add a _role_id_ column. The function for creating the _users_ table will then look like:-

```php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->tinyInteger('role_id'); //this is the new line
    $table->rememberToken();
    $table->timestamps();
});
```

- Modify the _UserSeeder.php_ file to include a _role_id_ for each user

```php
public function run(): void
{
    DB::table('users')->insert(['name' => 'Kate Hutton', 'email' => 'k.l.hutton@hud.ac.uk', 'password' => Hash::make('password'), 'role_id' => 2]);
    DB::table('users')->insert(['name' => 'Yousef Miandad', 'email' => 'y.miandad@hud.ac.uk', 'password' => Hash::make('letmein'), 'role_id' => 2]);
    DB::table('users')->insert(['name' => 'Sunil Laxman', 'email' => 's.laxman@hud.ac.uk', 'password' => Hash::make('password2'), 'role_id' => 1]);
}
```

- Re-run the migrations and seeding

```
php artisan migrate:fresh --seed
```

- Check Adminer to make sure this has worked.

### Adding a gate

- Add a gate to the `boot()` method of _app/Providers/ServiceProvider.php_.
  - A [gate](https://laravel.com/framework/docs/authorization#gates) is simply a function that decides if a user is allowed to perform a certain action. This gate simply checks if the user as a _role_id_ of 2.

```php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\Gate;
use App\Models\User;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        //
    }

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Gate::define('edit', function (User $user) {
            return $user->role_id === 2;
        });
    }
}
```

### Using the Gate

There are a number of different ways of using the gate - in a controller, a route, a view. We will use it in our routes file.

- Modify the _web.php_ so that only users that pass the _edit_ gate can access the create, store and edit routes.

```php
use App\Http\Controllers\FilmController;
use App\Http\Controllers\AuthController;
use Illuminate\Support\Facades\Route;

Route::get('/films', [FilmController::class, 'index']);
Route::get('/films/about', [FilmController::class, 'about']);
Route::get('/films/create', [FilmController::class, 'create'])->middleware(['auth', 'can:edit']);
Route::post('/films', [FilmController::class, 'store'])->middleware(['auth', 'can:edit']);
Route::get('/films/{id}', [FilmController::class, 'show'])->middleware('auth');
Route::get('/films/{id}/edit', [FilmController::class, 'edit'])->middleware(['auth', 'can:edit']);


Route::get('/login', [AuthController::class, 'index'])->name("login");
Route::post('/login', [AuthController::class, 'login']);
Route::post('/logout', [AuthController::class, 'logout']);
```

- Test this works
- Log into the app as s.laxman@hud.ac.uk (role_id of 1) and try to add a new film. You should be denied access.

### Using the @can Directive in Views

This works but really we want hide navigation options from users that don't have the authorisation to perform those actions.

- Open _layout.blade_.
- Modify the navigation bar to use the [`@can` blade directive](https://laravel.com/framework/docs/authorization#via-blade-templates).

```html
<nav>
  <ul>
    <li><a href="/films">Home</a></li>
    @can('edit')
    <li><a href="/films/create">Add new film</a></li>
    @endcan
    <li><a href="/films/about">About</a></li>
    @guest
    <li><a href="/login">Sign in</a></li>
    @endguest
  </ul>
</nav>
```

- Open _show.blade.php_ and hide the edit and delete buttons for non-authorised users.

```html
<x-layout title="Show the details for a film">
    <h1>{{$film->title}} ({{$film->certificate->name}})</h1>
    <p>Year:{{$film->year}}</p>
    <p>Duration:{{$film->duration}} ({{$film->getAge()}} years old)</p>

    @can('edit')
    <a href='/films/{{$film->id}}/edit'>
        <button>Edit</button>
    </a>

    <form method='POST' action='/films/{{$film->id}}'>
        @csrf
        @method('DELETE')
        <button type='submit'>Delete</button>
    </form>
    @endcan
</x-layout>
```

- Test this works and take a moment to understand the different levels of access for the app.

### Adding a custom 403 page

When a user isn't authorised to perform an action they receive a 403 page. It would be nice if this page was styled to fit with the rest of the application

- Create a new folder called 'errors' in the _resources/views_ folder (you may already have one).
- Create a new _403.blade.php_ file in this folder.
- Add the following code in this page.

```html
<x-layout title="You are not authorised to view this page">
    <h1>403</h1>
    <p>You are not authorised to view this page</p>
</x-layout>
```

- Test this works.

## Test your Understanding
- When the user enters the wrong login details they are sent back to the login form. It would be good if they received a message telling them the details they entered were incorrect. Have a look at https://laravel.com/docs/session#flash-data for advice on passing a message back to the login page.

## More to Investigate
- An alternative to using gates is using policies (https://laravel.com/docs/authorization#creating-policies), which give more control over access to actions.
