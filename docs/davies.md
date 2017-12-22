Here's an easy way to build a Magento2 site for you to play around in and learn how things work.

(This is based upon the Matt Blatt Development Environment procedure, but has been made more generic)



# You need to be in the www-data and docker groups, so you need to make sure the group exists, and that you're a member of it.
sudo groupadd -g 33 www-data
sudo usermod -a -G www-data $USER
sudo usermod -a -G docker $USER

# Make sure docker is running
service docker status
if [[ $? -ne 0 ]]; then sudo service docker restart; else echo "Docker seems to be ok :-)"; fi

# Clone the docker environment and the Matt Blatt code repository
cd ~/src
git clone git@github.com:mrda/docker-magento2.git
cd docker-magento2
mkdir magento

# Install docker-composer
# (Note: My suggestion is to install the latest python-pip first, because this will make your life must easier)
curl -O https://bootstrap.pypa.io/get-pip.py # You can skip this if you have a recent python-pip
sudo python get-pip.py                               # You can skip this if you have a recent python-pip
sudo pip docker-compose

# Collect credentials to set up composer; you'll use these in the next step:
# COMPOSER_GITHUB_TOKEN - source this from your github profile, settings, developer settings, personal access token, generate token
# COMPOSER_MAGENTO_USERNAME/COMPOSER_MAGENTO_PASSWORD - source this from https://marketplace.magento.com/customer/accessKeys/ ; log in with your magento.com account, generate access keys.  'public key' is the username, 'private key' is the password.

#Set up composer to run
composer config -g github-oauth.github.com <your github 'personal access token'>
cp composer.env.sample composer.env
vi composer.env
< Replace the long strings of zeros for:>
< COMPOSER_GITHUB_TOKEN with your github 'personal access token' obtained above>
< COMPOSER_MAGENTO_USERNAME and COMPOSER_MAGENTO_PASSWORD with the 'public key' and 'private key' respectively>

# Run the 'builder' script to copy config/scripts from src into php-7{0,1}-{cli,fpm} for docker-compose to pick up during build
docker run --rm -it -v $(pwd):/src php:7 php /src/builder.php

# Build the docker containers, so our local changes are included
docker-compose -f docker-compose-build.yml build

# Download and install Magento2 itself
docker-compose -f docker-compose-build.yml run cli magento-installer

# And start all the containers
docker-compose -f docker-compose-build.yml up -d

# If starting from scratch, set up the database schema

docker-compose -f docker-compose-build.yml exec fpm bash
cd ../magento
sudo -u www-data ./bin/magento setup:db-schema:upgrade
sudo -u www-data ./bin/magento setup:db-data:upgrade

# Validate magento is happy with it's database by running:
sudo -u www-data ./bin/magento setup:db:status


# Next step is to set up your new Magento2 installation for your ease of use.
# Jump into a shell on the PHP container, and set up an admin user
docker-compose -f docker-compose-build.yml exec fpm bash
cd ../magento
sudo -u www-data curl -O https://files.magerun.net/n98-magerun2.phar
sudo -u www-data chmod a+rx ./n98-magerun2.phar
sudo -u www-data ./n98-magerun2.phar admin:user:create --admin-user admin --admin-password al1g3nt --admin-email aligent@example.com --admin-firstname Aligent --admin-lastname Consulting

# Set the Magento Store URL
sudo -u www-data ./n98-magerun2.phar setup:store-config:set --base-url http://magento2.docker


# Optional: Add in the Luma sample data
./bin/magento sampledata:deploy # optional, if you want the sample data
./vendor/composer/composer/bin/composer update # you'll need the COMPOSER_MAGENTO_USERNAME and COMPOSER_MAGENTO_PASSWORD from your docker2-magento composer.env file
./bin/magento setup:upgrade
./bin/magento setup:static-content:deploy en_AU en_US
./bin/magento setup:di:compile
./bin/magento cache:flush

# Now exit the container shell
exit

# You're now back on your laptop, add a /etc/hosts file entry for your web container
IPADDR=$(docker-compose -f docker-compose-build.yml exec web ip route get 8.8.8.8| grep src| sed 's/.*src \(.*\)$/\1/g')
echo $IPADDR