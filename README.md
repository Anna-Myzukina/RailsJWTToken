### RailsJWTToken
## Background
Today I am going to give detailed steps on how to get knock gem up and running in your Rails application. The idea for this article stems from the fact that i couldn't find a well-defined and up to date resource on how to implement rails API token authentication. For this tutorial, I will be using the latest version (6.0) of Ruby on Rails.

## Setup

First thing first Generate a new rails app.
````ruby 
rails new JWTapp --api -d=postgresql -T
cd  JWTapp
rails db:create
rails db:migrate


````
## CORS
Let's uncomment Cors gem to permit access to the API.
````ruby 
gem 'rack-cors'
bundle install
````
Uncomment this lines below in cors.rb .
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
## Bcrypt & Knock Gem
Bcrypt is a hash algorithm for password hashing and knock which is mainly for JWT authentication.
````ruby 
gem 'bcrypt'
gem 'knock'
````
````ruby 
bundle install
````
## User Model & Controllers
Now we are going to create a user using the scaffold generator.

````ruby 
rails generate scaffold User username:string email:string password_digest:string

````
Include this in your user model.
````ruby 
# app/models/user.rb 
class User < ApplicationRecord
  has_secure_password
  validates :username, presence: true
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-]+(\.[a-z\d\-]+)*\.[a-z]+\z/i.freeze
  validates :email, presence: true,uniqueness: true, format: { with: VALID_EMAIL_REGEX }
end
````


The user controller generated by the scaffold gave us a field called `password_digest`. But Bcrypt, in addition to hashing the password, it also turns password_digest into two fields: `password` and `password_confirmation`. So we need to change the permitted params from password_digest to these two fields, also remove the  `location: @user` in the create method.

````ruby
def user_params
  params.permit(:username, :email, :password, :password_confirmation)
end

````
````ruby 
# users_controller.rb 
  def create
    @user = User.new(user_params)

    if @user.save
      render json: @user, status: :created
    else
      render json: @user.errors, status: :unprocessable_entity
    end
  end
````
````ruby 
rails db:migrate
````
Lets create a user using rails console
````ruby
rails c
````
````ruby
User.create(username: "Danny", email: "dan@ymail.com", password: "enemona", password_confirmation: "enemona")
````
## Knock Configuration
Now lets configure the knock Gem
````ruby 
rails g knock:install
````
This will create an initializer file at `config/initializers/knock.rb` that contains the default configuration. 

Yeah I know you got an error like this `Could not load generator "generators/knock/install_generator"`, if not skip to the next step(default knock token), this is caused by zeitwerk that is rails 6 autoloader.we can circumnavigate this error by switching the autoloader.

Read more about zeitwerk [here](https://medium.com/cedarcode/understanding-zeitwerk-in-rails-6-f168a9f09a1f).
Now we are going to degenerate the knock and generate it once more once we have included the line below into application.rb
````ruby 
rails d knock:install
````
 Lets include this in  application.rb file
````ruby 
# config/application.rb 
config.load_defaults 6.0 and config.autoloader = :classic
````
````ruby 
rails g knock:install
````
By default knock token is set to expire in 24 hours, we can adjust it if we uncomment the line below, and adjust it however we want.
````ruby 
# config/initializers/knock.rb 
config.token_lifetime = 1.day
````
Generate a controller for users to sign in through:

````ruby 
rails generate knock:token_controller user
````
This generates a controller called `user_token_controller`. It inherits from `Knock::AuthTokenController` which comes with a create action that will create a JWT when logged in.

Note:`user_token_controller` is meant for sign in and its different from `users_controller`.

````ruby 
# app/controllers/user_token_controller.rb
class UserTokenController < Knock::AuthTokenController
end

````
The generator also inserts a  route in the routes.rb file as an API endpoint for sign in.
 
````ruby 
# app/config/routes.rb
post 'user_token' => 'user_token#create'
````
 Now lets `include Knock::Authenticable` module in your ApplicationController.
````ruby 
# app/controllers/application_controller.rb 
class ApplicationController < ActionController::API
  include Knock::Authenticable
end
````
You almost there friend,Add an `authenticate_user` before filter to your controllers to be protected,here am going to first add it to `users_controller` and except the `create` method in order to enable users sign up.

````ruby 
# app/controllers/users_controller.rb 
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
Lastly, Rails no longer uses `config/secrets.yml` to hold the secret_key_base that is used for various security features, including generating JWTs with the Knock gem. Rails now uses an encoded file called `config/credentials.yml.enc`. Add the below line to the Knock configuration file
````ruby 
# config/initializers/knock.rb 
config.token_secret_signature_key = -> { Rails.application.credentials.secret_key_base }
  
````
# current_user
You also have access directly to `current_user` which will try to authenticate or return nil
 ````ruby 
def index
  if current_user
    # do something
  else
    # do something else
  end
end
````
### Routes
Add an api namespace using scopes,This time let's add an "auth/" scope to all of our api routes. That will add "auth" to the path but not to the controller or model.i edited the default route to `user_token controller`.
````ruby 

# config/routes.rb 
scope '/auth' do
  post '/signin', to: 'user_token#create'
  post '/signup', to: 'users#create'
end
````
## Testing
Now lets test the api using insomnia ,but you can use post man or Curl command line tool
Run the server,may the odds be on your side
````ruby 
rails s
````
````ruby 
{
  "auth": {
    "email": "dan@ymail.com",
    "password": "enemona"
  }
}
````
Note:`auth` is  an object containing the the log in form field names and values.
