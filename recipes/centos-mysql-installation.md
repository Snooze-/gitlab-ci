# WARING THIS DOCUMENT IS NOT COMPLETE YET AND FOLLOWING THE STEPS MIGHT GIVE YOU AN IDEA ON HOW TO GET THIS WORKING, BUT IS UNLIKELY TO RESULT IN A WORKING SYSTEM.

# Requirements:

* CentOS 6.4 Mimial Install

# Setup: 

## 1. Packages / Dependencies

**Note:**
Vim is an editor that is used here whenever there are files that need to be
edited by hand. But, you can use any editor you like instead.

    # Install vim
    sudo yum -y install vim-enhanced

### Add the EPEL repository

[EPEL][] is a volunteer-based community effort from the Fedora project to create
a repository of high-quality add-on packages that complement the Fedora-based
Red Hat Enterprise Linux (RHEL) and its compatible spinoffs, such as CentOS and Scientific Linux.

As part of the Fedora packaging community, EPEL packages are 100% free/libre open source software (FLOSS).

Install the `epel-release-6-8.noarch` package, which will enable EPEL repository on your system:

    yum install https://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm

Install the required packages:

    sudo yum -y update
    sudo yum -y groupinstall 'Development Tools'
    sudo yum -y install crontabs, curl, curl-devel, gcc, git, glibc-devel, libicu-devel, libxml2-devel, libxslt-devel, libyaml-devel, mysql-devel, mysql-server, openssh-server, openssl-devel, postfix, readline-devel, redis, wget

# 2. Ruby

Download Ruby and compile it:

    mkdir /tmp/ruby && cd /tmp/ruby
    curl --progress http://ftp.ruby-lang.org/pub/ruby/1.9/ruby-1.9.3-p448.tar.gz | tar xz
    cd ruby-1.9.3-p448/
    ./configure
    make
    make test
    sudo make install

Install the Bundler Gem:

    sudo gem install bundler --no-ri --no-rdoc

## 3. GitLab CI user:

    sudo adduser --system --shell /bin/bash --comment 'GitLab CI' --create-home --home-dir /home/gitlab_ci/ gitlab_ci

We do NOT set the password so this user cannot login.

## 4. Prepare the database

Although GitLab CI supporst PostgreSQL, this document has only been tested with MySQL. If you would like to try using PostgreSQL you can check the official installation document.

### MySQL

    # enable the `mysqld` service to start on boot:
    sudo chkconfig mysqld on
    sudo service mysqld start

    # Secure MySQL by entering a root password and say "Yes" to all questions:
    /usr/bin/mysql_secure_installation

    # Login to MySQL
    $ mysql -u root -p

    # Create the GitLab CI database
    mysql> CREATE DATABASE IF NOT EXISTS `gitlab_ci_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;

    # Create the MySQL User change $password to a real password
    mysql> CREATE USER 'gitlab_ci'@'localhost' IDENTIFIED BY '$password';

    # Grant proper permissions to the MySQL User
    mysql> GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER ON `gitlab_ci_production`.* TO 'gitlab_ci'@'localhost';

    # Flush MySQL privileges
    mysql> FLUSH PRIVILEGES

    # Logout MYSQL
    mysql> exit;

## 5. Get code 

    cd /home/gitlab_ci/

    sudo -u gitlab_ci -H git clone https://github.com/gitlabhq/gitlab-ci.git

    cd gitlab-ci

## 6. Setup application

    # Edit application settings
    sudo -u gitlab_ci -H cp config/application.yml.example config/application.yml
    sudo -u gitlab_ci -H vim config/application.yml

    # Edit web server settings
    sudo -u gitlab_ci -H cp config/puma.rb.example config/puma.rb
    sudo -u gitlab_ci -H vim config/puma.rb

    # Create socket and pid directories
    sudo -u gitlab_ci -H mkdir -p tmp/sockets/
    sudo chmod -R u+rwX  tmp/sockets/
    sudo -u gitlab_ci -H mkdir -p tmp/pids/
    sudo chmod -R u+rwX  tmp/pids/

### Install gems
 
    # For MySQL (note, the option says "without ... postgres")
    su - gitlab_ci
    cd gitlab-ci/
    bundle install --without development test postgres --deployment

### Setup db

    # mysql
    cp config/database.yml.mysql config/database.yml
 
    # Edit user/password
    vim config/database.yml

    # Setup tables
    bundle exec rake db:setup RAILS_ENV=production

    # Setup schedules
    bundle exec whenever -w RAILS_ENV=production
    exit

## 7. Install Init Script

Download the init script (will be /etc/init.d/gitlab_ci):

    sudo wget https://raw.github.com/gitlabhq/gitlab-ci/master/lib/support/init.d/gitlab_ci -P /etc/init.d/
    sudo chmod +x /etc/init.d/gitlab_ci

Make GitLab start on boot:

    sudo chkconfig --level 345 gitlab_ci on

Start your GitLab instance:

    service gitlab_ci restart

# 8. Nginx


## Installation
    sudo yum -y install nginx

## Site Configuration

Download an example site config:

    sudo wget https://raw.github.com/gitlabhq/gitlab-ci/master/lib/support/nginx/gitlab_ci -P /etc/nginx/sites-available/
    sudo ln -s /etc/nginx/sites-available/gitlab_ci /etc/nginx/sites-enabled/gitlab_ci

Make sure to edit the config file to match your setup:

    # Change **YOUR_SERVER_IP** and **YOUR_SERVER_FQDN**
    # to the IP address and fully-qualified domain name
    # of your host serving GitLab CI
    sudo vim /etc/nginx/sites-enabled/gitlab_ci

Add nginx to the gitlab_ci group and set group permissions on the /home/gitlab_ci directory

    usermod -aG gitlab_ci nginx
    chmod g+rx /home/gitlab_ci/

## Make GitLab start on boot

    sudo chkconfig --level 345 nginx on

## Reload configuration

    sudo service nginx restart

# 9. Runners


Now you need Runners to process your builds.
Checkout [runner repository](https://github.com/gitlabhq/gitlab-ci-runner#installation) for setup info.

# Done!


Visit YOUR_SERVER for your first GitLab CI login.
You should use your GitLab credentials in order to login

**Enjoy!**
