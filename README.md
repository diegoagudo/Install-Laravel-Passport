
# Installing Laravel Passport
 [![HOW-TO](https://img.shields.io/badge/HOW--TO-Installing%20Laravel%20Passport-red)](https://github.com/diegoagudo/OAuth2Client.js/blob/master/LICENSE)

 Laravel Passport provides a full OAuth2 server implementation for your Laravel application in a matter of minutes. Passport is built on top of the [League OAuth2 server](https://github.com/thephpleague/oauth2-server) that is maintained by Andy Millington and Simon Hamp.

## Requirements
* [Laravel](https://www.laravel.com)

## Installation
To get started, install Passport following [official installation guide](https://laravel.com/docs/8.x/passport#installation)

##### Remember
* Run the command `install` with option `--uuids`. This option will instruct Passport that you would like to use UUIDs instead of auto-incrementing integers
```php
php artisan passport:install --uuids
```

## Post Install
### Step 1
Create a custom migrate for table *oauth_clients* to add *redirect_logout* field. 
This field will be necessary to validate user logout callback.
```PHP
	php artisan make:migration alter_oauth_clients
```
and replace the content for
```php
	<?php  
	use Illuminate\Database\Migrations\Migration;  
	use Illuminate\Database\Schema\Blueprint;  
	use Illuminate\Support\Facades\Schema;  
	  
	class AlterOauthClients extends Migration  
	{  
	  /**  
	 * Run the migrations. * * @return void  
	 */  
	 public function up() { 
		 Schema::table('oauth_clients', function (Blueprint $table) {  
			  $table->text('redirect_logout')->nullable();  
		  });  
	  }  
	  
	  /**  
	 * Reverse the migrations. * * @return void  
	 */  
	 public function down() {
		  Schema::table('oauth_clients', function (Blueprint $table) {  
			  $table->dropColumn('redirect_logout');  
		  });  
	  }  
	}
```
Now, execute the migration to make effect
```php
	php artisan migrate
```

### Step 2
Create a custom `PassportClient` model to skip authorization prompt:
```php
<?php  
	namespace App\Models;  
	use Laravel\Passport\Client;  
	  
	class PassportClient extends Client  
	{  
	  
	 /**  
	  * Determine if the client should skip the authorization prompt. 
	  ** @return bool  
	  */ 
	  public function skipsAuthorization() {
	    return true;  
	  }  
	}
```

### Step 3
Next, you should call the custom `PassportClient` model  method within the `boot` method of your `App\Providers\AuthServiceProvider`. This method will register custom model:
```php
<?php

namespace App\Providers;

use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;
use Illuminate\Support\Facades\Gate;
use Laravel\Passport\Passport;
use App\Models\PassportClient;

class AuthServiceProvider extends ServiceProvider
{
    /**
     * The policy mappings for the application.
     *
     * @var array
     */
    protected $policies = [
        'App\Models\Model' => 'App\Policies\ModelPolicy',
    ];

    /**
     * Register any authentication / authorization services.
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

		Passport::useClientModel(PassportClient::class);
				
        if (! $this->app->routesAreCached()) {
            Passport::routes();
        }
    }
}
```


### Step 4
Laravel Passport not authorize new users, so edit `RegisterController.php` and put this lines:
```php
	protected function registered(Request $request)  
	{  
	  if ($request->session()->has('url.intended')) {  
		  return redirect()->intended();  
	  }  
	}
```
When register step was finished, will redirect to `intended URL` that's contains information about authorization.

### Step 5
Instead use the command `php artisan passport:client`, will create a command with custom parameters:
```php
	<?php
	namespace App\Console\Commands;

	use Illuminate\Console\Command;
	use Illuminate\Support\Str;
	use Laravel\Passport\Passport;

	class PassportClient2 extends Command
	{
	    /**
	     * The name and signature of the console command.
	     *
	     * @var string
	     */
	    protected $signature = 'passport:clientv2';

	    /**
	     * The console command description.
	     *
	     * @var string
	     */
	    protected $description = 'Create a client v2 for issuing access tokens';

	    /**
	     * Create a new command instance.
	     *
	     * @return void
	     */
	    public function __construct()
	    {
	        parent::__construct();
	    }

	    /**
	     * Execute the console command.
	     *
	     * @return int
	     */
	    public function handle()
	    {
	        $input = [];
	        $input['name'] = $this->ask('Qual o nome do "Client" ou Aplicação que irá se integrar?');
	        $input['redirect'] = $this->ask('Informe a URL de redirecionamento após o Login');
	        $input['redirect_logout'] = $this->ask('Informe a URL de redirecionamento após o Logout');

	        if (!filter_var($input['redirect'], FILTER_VALIDATE_URL)) {
	            $this->error('Formato da URL de Login inválido.');
	            return;
	        }
	        if (!filter_var($input['redirect_logout'], FILTER_VALIDATE_URL)) {
	            $this->error('Formato da URL de Logout inválido.');
	            return;
	        }

	        $clientSecret = Str::random(40);
	        $client = Passport::client()->forceFill([
	            'name' => $input['name'] ?? null,
	            'secret' => $clientSecret,
	            'redirect' => $input['redirect'],
	            'redirect_logout' => $input['redirect_logout'],
	            'personal_access_client' => 0,
	            'password_client' => 0,
	            'revoked' => false,
	        ]);

	        $client->save();

	        $this->info('********************************');
	        $this->info('* Aplicação criada com sucesso *');
	        $this->info(' ');
	        $this->info(' Client Id '. $client->id);
	        $this->info(' Client Secret: '. $clientSecret);
	        $this->info(' Login Redirect URL: '. $input['redirect']);
	        $this->info(' Logout Redirect URL '. $input['redirect_logout']);
	        $this->info(' ');
	        $this->info(' Lembre-se de utilizar a URLs informadas, caso contrário a requisição será negada. ');
	        $this->info('********************************');

	        return Command::SUCCESS;
	    }
	}
```

### Step 6
Edit the file `routes.php` and create route for logout:
```php
	Route::get('/oauth/logout', function(\Illuminate\Http\Client\Request $request) {  
	  $oauthClients = Illuminate\Support\Facades\DB::table('oauth_clients')  
	 ->where('id', $request->client_id)  
	 ->where('revoked', 0)  
	 ->first();  
	  
	 if(!$oauthClients)  
	  die('Bad request');  
	  
	 if(!$request->redirect_url_logout OR $request->redirect_url_logout != $oauthClients->redirect_logout)  
	  die('Bad request');  
	  
	  Illuminate\Support\Facades\Auth::logout();  
	  
	 return redirect($request->redirect_url_logout);  
	});
```
