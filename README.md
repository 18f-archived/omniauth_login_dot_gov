# login.gov Omniauth strategy

This gem is an Omniauth strategy to provide authentication with Login.gov in a rack application with the OpenID:Connect protocol.

#### ⚠️  Common Vulnerabilities and Exposure Warning:
There is a known vulnerability with Omniauth that affects this gem as
well as any implementation of Omniauth with a single strategy. Please
review [CVE-2015-9284](https://nvd.nist.gov/vuln/detail/CVE-2015-9284) for more
information and mitigation steps. The [Omniauth Wiki](https://github.com/omniauth/omniauth/wiki/Resolving-CVE-2015-9284) also
has relevant information (August 2019).

The vulnerability allows for an attacker to connect an authentication provider (like Login.gov) to a client application without any user interaction. The vulnerability is partially mitigated for this gem because Login.gov requires users to explicitly authorize an application on Login.gov before an application is connected. Users of this gem should still understand the vulnerability and mitigate further with the steps detailed in the Omniauth wiki above.

## Getting started in a Rails app

Our [developer documentation](https://developers.login.gov/oidc/) will have the most up-to-date information on OIDC integration including available scopes and attributes.

#### Generate your keys

Before you begin, you will need a self-signed private and public certificate/key pair.
To generate the private and public key pair and output your public certificate with PEM formatting:

```shell
openssl req -nodes -x509 -days 365 -newkey rsa:2048 -keyout private.pem -out public.crt
```

Note: the `-nodes` flag skips encryption of the private key and is not recommended for production use.

#### Register your app with the login.gov sandbox

You will need to register your service provider with our sandbox environment using the login.gov [Partner Dashboard](https://dashboard.int.identitysandbox.gov). Note: this is not your production login.gov account.

1. Log in to the [Partner Dashboard](https://dashboard.int.identitysandbox.gov).
2. Choose [Create new test app](https://dashboard.int.identitysandbox.gov/service_providers/new).
3. Enter the required fields to identify your service provider, including the **public** self-signed certificate you generated earlier.

#### Configure your app

Now that your app is configured with login.gov's sandbox as a Service Provider, you can begin configuring your local application.

1. Make the private key available to your application. This can be done by running `bin/rails credentials:edit` and adding the following:
  ```yaml
  login_pem: |-
    -----BEGIN PRIVATE KEY-----
    <PRIVATEKEY>
    -----END PRIVATE KEY-----
  ```
2. Add this gem to the Gemfile:
  ```ruby
  gem 'omniauth_login_dot_gov', :git => 'https://github.com/18f/omniauth_login_dot_gov.git',
                                :branch => 'main'
  ```
3. Additionally, add the Omniauth CSRF protection gem to the Gemfile to address the CVE referenced above:
  ```ruby
  gem 'omniauth-rails_csrf_protection'
  ```
4. Install these gems and dependencies with `bundle install`
5. Now, configure the Omniauth middleware with an initializer:
  ```ruby
  # config/initializers/omniauth.rb
  Rails.application.config.middleware.use OmniAuth::Builder do
    provider :login_dot_gov, {
      name: :login_dot_gov,
      client_id: 'urn:gov:gsa:openidconnect:sp:myapp', # same value as registered in the Partner Dashboard
      idp_base_url: 'https://idp.int.identitysandbox.gov/', # login.gov sandbox environment IdP
      ial: 1,
      private_key: OpenSSL::PKey::RSA.new(Rails.application.credentials.login_pem),
      redirect_uri: 'http://localhost:3000/auth/logindotgov/callback',
    }
  end
  ```
6. Create a controller for handling the callback, such as this:
  ```ruby
  # app/controllers/users/omniauth_controller.rb
  module Users
    class OmniauthController < ApplicationController
      def callback
        omniauth_info = request.env['omniauth.auth']['info']
        @user = User.find_by(email: omniauth_info['email'])
        if @user
          @user.update!(uuid: omniauth_info['uuid'])
          sign_in @user
          redirect_to service_providers_path

        # Can't find an account, tell user to contact login.gov team
        else
          redirect_to users_none_url
        end
      end
    end
  end
  ```
7. Add the callback route to `routes.rb`
  ```ruby
  get '/auth/login_dot_gov/callback' => 'users/omniauth#callback'
  ```

8. Start your application and send a `POST` request to: `/auth/login_dot_gov` (eg. http://localhost:3000/auth/login_dot_gov) to initiate authentication with login.gov!


## Public domain

This project is in the worldwide [public domain](LICENSE.md). As stated in [CONTRIBUTING](CONTRIBUTING.md):

> This project is in the public domain within the United States, and copyright and related rights in the work worldwide are waived through the [CC0 1.0 Universal public domain dedication](https://creativecommons.org/publicdomain/zero/1.0/).
>
> All contributions to this project will be released under the CC0
> dedication. By submitting a pull request, you are agreeing to comply
> with this waiver of copyright interest.
