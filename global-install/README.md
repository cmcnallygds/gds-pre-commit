# Global install 

## Rationale

* We want to run this code against all commits in alphagov.
* We want to know who is using the pre-commits. 
* We want to know how many alphagov repos are protected. 

## Idea 

We have an install script possibly wrapped up in GDS cli which registers 
users and installs the hooks into their global git config rather than 
installing per repository. 

## How registration works 

The install script interrogates their existing global config 
`git config --global --list`. 

The `user.name` in git config is not your github username so we still 
need that.

### With SSH

We find the SSH key by interrogating their `.ssh/config` or falling 
back on `id_rsa`  

It identifies their github username by doing something like 
`ssh -T git@github.com` and parsing the name from the response

>  Hi [username]! You've successfully authenticated, but GitHub does not provide shell access.

This would all have to be interactive to enable users with SSH passwords 
or YubiKeys.

### With a PAT 

We can't do 
`curl -u username:token https://api.github.com/user` because you need 
to know the username.

Then we generate some random secret from that content - an MD5 would be 
fine - and sign it using the users private key to create a B64 string. 

This is then set in the user's global config:
`git config --global gds-cyber.bearer TOKEN`

We send a message to the alert controller using this as a bearer token 
in the request header sending both the md5 and the username and context.

We use `https://api.github.com/search/users/[username]/keys` to get 
their public keys and ensure we can verify the signed request. 

If we can we store the secret (and the public key?) in SSM to auth future
requests. 

## How usage reporting works

We add our own pre-commit hook into the list from this repo which posts 
to the alert-controller using the TOKEN. 

We have a lambda which validates the user and plumbs the data through to 
splunk.

## Questions / Issues 

* A lot of people's github accounts are not attached to their GDS email
* Do we need to confirm membership of alphagov? 
* What about non-SSH based access - PATs?
* What about key rotation?
* Can we tell whether the other pre-commit hooks passed/failed 
    and report that?
    
## Make it simpler 

* We could just ask people to enter their github username in the install 
step and save it to git global config.
