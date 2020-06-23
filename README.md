### RailsJWTToken
First thing first Generate a new rails api
````ruby 
rails new JWTapp --api -d=postgresql
````
### Add cors to permit access to the api
````ruby 
gem 'rack-cors'
bundle install
````
### Uncomment this lines below in cors.rb
````ruby 
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins 'http://localhost:3000'

    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end
````

### Bcrypt is hash algorithm  for password hashing and knock which is mainly for JWT authentication
````ruby 
gem 'bcrypt'
gem 'knock'
bundle install
````
### Now we are going to generate User

````ruby 
rails generate scaffold User username:string email:string password_digest:string
# app/models/user.rb 
class User < ApplicationRecord
  has_secure_password
  validates :username, presence: true
  validates :email, presence: true, uniqueness: true
end
````


The users controller generated by the scaffold gave us a field called password_digest, just like we told it to. But Bcrypt, in addition to hashing the password, it also turns password_digest into two fields: password and password_confirmation. So we need to change the permitted params from :password_digest to those two fields:
````ruby 
# users_controller.rb 
def user_params
  params.permit(:username, :email, :password, :password_confirmation)
end

rails db:migrate
````
### Lets create a user
````ruby
rails c
User.create(username: "Daniel Adama", email: "dan@ymail.com", password: "enemona", password_confirmation: "enemona")
````
### Now lets configure the knock Gem
````ruby 
rails generate knock:install
````
This will create an initializer file at config/initializers/knock.rb that contains the default configuration. 

Note:you might get an error like this `Could not load generator "generators/knock/install_generator"` this caused by zeitwerk that is rails 6 autoloader.we can circumnavigate this error by switching the autoloader.
Read more about zeitwerk [here](https://medium.com/cedarcode/understanding-zeitwerk-in-rails-6-f168a9f09a1f) 
### Lets include this in  application.rb file
````ruby 
# config/application.rb 
config.load_defaults 6.0 and config.autoloader = :classic
````

By default knock token is set to expire in 24 hours, we can adjust it if we uncomment the line below ,and adjust it however we want 
````ruby 
# config/initializers/knock.rb 
config.token_lifetime = 1.day
````
### Generate a controller for users to login in through:
````ruby 
rails generate knock:token_controller user
````
This generates a controller called user_token_controller. It inherits from Knock::AuthTokenController which comes with a create action that will create a JWT when logged in. The generator also inserts a route in the routes.rb file: post 'user_token' => 'user_token#create' as an API endpoint for login.
### Now lets include Knock::Authenticable module in your ApplicationController 
````ruby 
# app/controllers/application_controller.rb 
class ApplicationController < ActionController::API
  include Knock::Authenticable
end
````
poooo u almost there friend,Add an authenticate_user before filter to any controller to be authenticated,here am going to first add it to users_controller in order to enable users signup.
````ruby 
# app/controllers/user_controller.rb 
before_action :authenticate_user,except: [:create]

````
If you are using Rails 5.2 or higher you need to take two more steps.
- First,since protect_from_forgery is included in ActionController::Base by default, now you need to skip that in the Knock controller we generated for logging in from our React client. API clients are from a different domain so they won't have the standard Rails authenticity token.

````ruby 
# app/controllers/user_token_controller.rb
class UserTokenController < Knock::AuthTokenController
  skip_before_action :verify_authenticity_token, raise: false
end  
````
Lastly, Rails no longer uses config/secrets.yml to hold the secret_key_base that is used for various security features, including generating JWTs with the Knock gem. Rails now uses an encoded file called config/credentials.yml.enc. Add the below line to the Knock configuration file
````ruby 
# config/initializers/knock.rb 
config.token_secret_signature_key = -> { Rails.application.credentials.secret_key_base }
  
````
Add an api namespace using scopes
This time let's add an "auth/" scope to all of our api routes. That will add "auth" to the path but not to the controller or model.
````ruby 

# config/routes.rb 
scope '/auth' do
  post '/signin', to: 'user_token#create'
  post '/signup', to: 'user#create'
end
````
### Now lets test the api using post man

