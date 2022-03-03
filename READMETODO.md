https://enlear.academy/how-to-create-a-rails-6-api-with-devise-jwt-46fa35085e85


# Devise API Authentication With Vue JS | Ruby on Rails 7 Tutorial - YouTube - Deanin

Date: February 28, 2022 4:01 PM
Tags: Authentification, Backend, Devise, Dév, Learning, Rails, Vidéo, Web, Youtube
URL: https://www.youtube.com/watch?v=PqizV5l1yFE

![https://i.ytimg.com/vi/PqizV5l1yFE/maxresdefault.jpg](https://i.ytimg.com/vi/PqizV5l1yFE/maxresdefault.jpg)

---

## Creating the Devise API

1. Create Rails App

`rails new my_api --api`

2. Add required gems

`bundle add devise devise-jwt rack-cors`

3. Changements CORS

```ruby
# Be sure to restart your server when you modify this file.

# Avoid CORS issues when API is called from the frontend app.
# Handle Cross-Origin Resource Sharing (CORS) in order to accept cross-origin AJAX requests.

# Read more: https://github.com/cyu/rack-cors

Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins '*'

    resource '*',
             headers: :any,
             methods: %i[get post put patch delete options head],
             expose: %w[Authorization Uid]
  end
end
```

4. Installation de Devise

`rails g devise:install`

`rails g devise User`

## Devise JWT Tutorial

1. Génération de la Denylist *(sert pour le logout de l’utilisateur)*

`rails g model jwt_denylist jti:string exp:datetime`

<aside>
⚠️ Ne pas oublier de renommer la migration, la classe et la table pour les mettre au singulier !

</aside>

Fichier de migration :

```ruby
class CreateJwtDenylist < ActiveRecord::Migration[7.0]
  def change
    create_table :jwt_denylist do |t|
      t.string :jti, null: false
      t.datetime :exp, null: false

      t.timestamps
    end
    add_index :jwt_denylist, :jti
  end
end
```

2. Ajouts au model User

```ruby
class User < ApplicationRecord
	# Il faut ajouter les deux modules commençant par jwt
	devise :database_authenticatable, :registerable,
	:jwt_authenticatable,
	jwt_revocation_strategy: JwtDenylist
end
```

3. Petits ajouts des familles au Model `JwtDenylist`

```ruby
class JwtDenylist < ApplicationRecord
  include Devise::JWT::RevocationStrategies::Denylist

  self.table_name = 'jwt_denylist'
end
```

4. `rails db:migrate` 🙂

## Devise API JWT Controllers for Sessions and Registrations

1. Créer le fichier `members_controller.rb`

```ruby
# app/controllers/members_controller.rb
class MembersController < ApplicationController
  before_action :authenticate_user!

  def show
    user = get_user_from_token
    render json: {
      message: "If you see this, you're in!",
      user: user
    }
  end

  private

  def get_user_from_token
    jwt_payload = JWT.decode(request.headers['Authorization'].split(' ')[1],
                             Rails.application.credentials.devise[:jwt_secret_key]).first
    user_id = jwt_payload['sub']
    User.find(user_id.to_s)
  end
end
```

2. Création des Users

Deux nouveaux controllers à créer, qui modifieront les controllers de registration et de session de Devise

```ruby
# app/controllers/users/registrations_controller.rb
class Users::RegistrationsController < Devise::RegistrationsController
  respond_to :json

  private

  def respond_with(resource, _opts = {})
    register_success && return if resource.persisted?

    register_failed
  end

  def register_success
    render json: {
      message: 'Signed up sucessfully.',
      user: current_user
    }, status: :ok
  end

  def register_failed
    render json: { message: 'Something went wrong.' }, status: :unprocessable_entity
  end
end
```

```ruby
# app/controllers/users/sessions_controller.rb
class Users::SessionsController < Devise::SessionsController
  respond_to :json

  private

  def respond_with(_resource, _opts = {})
    render json: {
      message: 'You are logged in.',
      user: current_user
    }, status: :ok
  end

  def respond_to_on_destroy
    log_out_success && return if current_user

    log_out_failure
  end

  def log_out_success
    render json: { message: 'You are logged out.' }, status: :ok
  end

  def log_out_failure
    render json: { message: 'Hmm nothing happened.' }, status: :unauthorized
  end
end
```

## Devise JWT Secret Key

1. Config du secret JWT utilisé pour décoder les tokens dans `config/initializers/devise.rb` 

```ruby
Devise.setup do |config|
	# Plein de code
	config.jwt do |jwt|
		jwt.secret = Rails.application.credentials.devise[:jwt_secret_key]
	end
	# Encore tout plein de code
end
```

2. Génération du secret
    - `rake secret`
    - Copie du secret généré
    - `EDITOR=nano rails credentials:edit`
    - Ajout en bas du fichier de :
    
    ```bash
    devise:
      jwt_secret_key: [clé copiée] // ⚠ Il faut mettre 2 espaces au début de cette ligne
    ```
    

## Routing for Devise API

1. Go `config/routes.rb`

```ruby
Rails.application.routes.draw do
  devise_for :users,
             controllers: {
               sessions: 'users/sessions',
               registrations: 'users/registrations'
             }
  get '/member-data', to: 'members#show'
  # Define your application routes per the DSL in https://guides.rubyonrails.org/routing.html

  # Defines the root path route ("/")
  # root "articles#index"
end
```

## Configure Session Store in Rails 7

1. Config pour utiliser les cookies dans `config/application.rb`

```ruby
module DeviseVue
  class Application < Rails::Application
    # Du code cool

    # This also configures session_options for use below
    config.session_store :cookie_store, key: '_interslice_session'

    # Required for all session management (regardless of session_store)
    config.middleware.use ActionDispatch::Cookies

    config.middleware.use config.session_store, config.session_options

    # Plein de code
  end
end
```

## Et les routes c’est quoi ?

### Register

`POST /users`

Données attendues :

```json
{
	"user": {
		"email": string,
		"password": string
	}
}
```

### Login

`POST /users/sign_in`

Données attendues

```json
{
	"user": {
		"email": string,
		"password": string
	}
}
```

### Logout

`DELETE /users/sign_out`

Authentification nécessaire

### Login with token

`GET /member-data`

Authentification nécessaire
