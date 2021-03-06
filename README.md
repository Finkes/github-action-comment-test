# Workaround: Write Access for GitHub Actions in PRs

This repository demonstrates how to use a GitHub App (Bot) to get write access to certain parts of the GitHub API inside a GitHub Actiom. Due to security reasons GitHub Actions currently only have read access to the API for pull requests. This is really annoying if want to post a comment from an Action for PR since this is one of my favourite use cases of actions.

The idea of this workaround is that you register your own GitHub App which will have a bot identity. You'll be able to specify exactly which scopes you needed! 

This is only a workaround and I still hope that there will be a better solution available soon!




1. create a github app here https://github.com/settings/apps/new
```
Name: <name of your bot>
Homepage URL: You can use the URL to your repo

Permissions:

Pull requests (read & write)
Issues? <- I used it because PR comments are actually sent via the issue comments API
Metadata (read only) [mandatory by default]
```

2. click on create App

3. (optional) Upload an image for your app. This image will be shown for your bot user when creating comments.

4. create private key by clicking **Generate a private key** and save the file on your computer. You'll later have to copy the content of this file into a secret of your GitHub Action.

5. Install your GitHub App into your org. 

6. Generate an JWT token for your App as decribed by [Authenticating as GitHub App](https://developer.github.com/apps/building-github-apps/authenticating-with-github-apps/#authenticating-as-a-github-app)

This is an example for ruby. You'll need to enter your **APP ID** and the **path to the secret key file**:
```rb
require 'openssl'
require 'jwt'  # https://rubygems.org/gems/jwt

# Private key contents
private_pem = File.read(YOUR_PATH_TO_PEM)
private_key = OpenSSL::PKey::RSA.new(private_pem)

# Generate the JWT
payload = {
  # issued at time
  iat: Time.now.to_i,
  # JWT expiration time (10 minute maximum)
  exp: Time.now.to_i + (10 * 60),
  # GitHub App's identifier
  iss: YOUR_APP_ID
}

jwt = JWT.encode(payload, private_key, "RS256")
puts jwt
```
To try this code locally you need to install the jwt package:
```
sudo gem install jwt
```

To execute the snippet:
```
ruby my_file.rb
```
The script will then print the **JWT Token** on the command line.

The following steps make use of the GitHub Apps API to generate a token for your installation. This token can be used to post comments on a PR. https://developer.github.com/v3/apps/

7. Use the github apps API to get your installation. You need it to find the **installation ID**:


```
GET https://api.github.com/app/installations 
HEADERS:
  Authorization: Bearer <your generated JWT token>
  Accept: application/vnd.github.machine-man-preview+json
```

8. create an installation token via the API:
```
POST https://api.github.com/app/installations/:installation_id/access_tokens
HEADERS:
  Authorization: Bearer <your generated JWT token>
  Accept: application/vnd.github.machine-man-preview+json
```

9. Use the token to post a comment to your PR:

```
POST https://api.github.com/repos/:org/:repo/issues/:ID_OF_PR/comments
HEADERS:
  Authorization: Bearer <your installation token>

BODY (JSON):
{
  "body": "hello from github action"
}
```


To use this mechanism inside a GitHub action I hope that you can simply do steps 6-9 on the fly for every run of your action. The only requirement will be that you store the generated key in a **GitHub Secret**.

