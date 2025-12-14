# WordPress Development Workflow Guide

## Overview

This guide will walk you through setting up a professional WordPress development environment on your Mac. Here's what you'll accomplish:

1. **Set up a local development environment** with Apache, PHP, and MySQL
2. **Install and configure WordPress** for local development
3. **Set up Git version control** to track your changes
4. **Configure virtual hosts** to use custom domain names locally
5. **Enable port forwarding** to access your site without specifying a port
6. **Configure WordPress 6.9 with TwentyTwentyFive theme** using the native Block Editor and Site Editor
7. **Create staging and production environments** for testing and deployment
8. **Establish a workflow** for moving changes between environments

The process is designed to be modular - you can start with the local setup and add the additional components as your project grows.

## Table of Contents

1. [Local Development Environment Setup](#local-development-environment-setup)
2. [WordPress Installation](#wordpress-installation)
3. [Git Version Control Setup](#git-version-control-setup)
4. [Remote Repository Setup](#remote-repository-setup)
5. [Staging Environment Configuration](#staging-environment-configuration)
6. [Production Environment Setup](#production-environment-setup)
7. [Database Synchronization](#database-synchronization)
8. [Site Editor and Theme Configuration](#site-editor-and-theme-configuration)
9. [Development and Deployment Workflow](#development-and-deployment-workflow)
10. [Collaboration Guide](#collaboration-guide)
11. [Troubleshooting](#troubleshooting)

## Local Development Environment Setup

### Step 1: Install Required Software

First, let's install all the necessary software for your local development environment.

```bash
# Open Terminal and install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Make sure Homebrew is up to date
brew update

# Install PHP
brew install php

# Install MySQL
brew install mysql

# Install Apache
brew install httpd

# Start MySQL service
brew services start mysql

# Start Apache service
brew services start httpd

# Install WordPress CLI for easy management
brew install wp-cli

# Install Git if not already installed
brew install git
```

### Step 2: Set Up MySQL

Now let's set up MySQL with a secure password and create our database.

```bash
# Secure MySQL installation
mysql_secure_installation
```

Follow the prompts to set a root password and answer "Y" to all security questions.

```bash
# Log in to MySQL to create a database and user for WordPress
mysql -u root -p
```

Once logged in, create the database and user:

```sql
CREATE DATABASE databasename;
CREATE USER "wordpressusername"@"localhost" IDENTIFIED BY "your_strong_password";
GRANT ALL PRIVILEGES ON databasename.* TO "wordpressusername"@"localhost";
FLUSH PRIVILEGES;
EXIT;
```

Replace `your_strong_password` with a secure password.

### Step 3: Configure Apache

Let's configure Apache to work with PHP and enable user directories for WordPress development.

```bash
# Get the Homebrew installation path
BREW_PREFIX=$(brew --prefix)

# Open the Apache configuration file
nano $BREW_PREFIX/etc/httpd/httpd.conf
```

Find and uncomment (remove the # at the beginning) these lines:
```
LoadModule rewrite_module lib/httpd/modules/mod_rewrite.so
LoadModule userdir_module lib/httpd/modules/mod_userdir.so
```

PHP support needs to be enabled manually. Add this at the end of the DSO Support section:
```
# Manually added PHP support
LoadModule php_module /opt/homebrew/opt/php/lib/httpd/modules/libphp.so

<IfModule php_module>
    <FilesMatch \.php$>
         SetHandler application/x-httpd-php
    </FilesMatch>
</IfModule>
```

Find the line that sets the listening port (Note: we're using port 8080 to avoid permission issues with privileged ports):
```
Listen 8080
```

Find the `DirectoryIndex` directive and make sure it includes index.php:
```
DirectoryIndex index.php index.html
```

Find the `DocumentRoot` and `Directory` sections and update them:
```
DocumentRoot "$BREW_PREFIX/var/www"
<Directory "$BREW_PREFIX/var/www">
    AllowOverride All
    Require all granted
</Directory>
```
Strictly speaking this is not necessary since we're using user directory support. All websites are in `~/Sites`, which also avoids having to use superuser privileges.

At the end of the file enable user directories and virtual hosts:
```
# User home directories
Include $BREW_PREFIX/etc/httpd/extra/httpd-userdir.conf

# Include virtual hosts
Include $BREW_PREFIX/etc/httpd/vhosts/httpd-vhosts.conf
```

Now, we need to configure the user directory and virtual hosts modules.

### Step 4: Configure User Directory

```
# Settings for user home directories
#
# Required module: mod_authz_core, mod_authz_host, mod_userdir

#
# UserDir: The name of the directory that is appended onto a user's home
# directory if a ~user request is received.  Note that you must also set
# the default access control for these directories, as in the example below.
#
#UserDir public_html
UserDir Sites

#
# On Mac OS and for WordPress sites we need more permissions than read-only.
#
<Directory "/Users/*/Sites">
    AllowOverride All
    Options Indexes MultiViews FollowSymLinks
    Require all granted
</Directory>
```

### Step 5: Configure Virtual Hosts

```
# Virtual Hosts
#
# Required modules: mod_log_config

# If you want to maintain multiple domains/hostnames on your
# machine you can setup VirtualHost containers for them. Most configurations
# use only name-based virtual hosts so the server doesn't need to worry about
# IP addresses. This is indicated by the asterisks in the directives below.
#
# Please see the documentation at 
# <URL:http://httpd.apache.org/docs/2.4/vhosts/>
# for further details before you try to setup virtual hosts.
#
# You may use the command line option '-S' to verify your virtual host
# configuration.

#
# VirtualHost example:
# Almost any Apache directive may go into a VirtualHost container.
# The first VirtualHost section is used for all requests that do not
# match a ServerName or ServerAlias in any <VirtualHost> block.
#

<VirtualHost *:8080>
    ServerAdmin webmaster@zsell.ai
    ServerName zsell.ai.local
    DocumentRoot "/Users/wernerheigl/Sites/zsell-ai-wordpress"
    ErrorLog "/opt/homebrew/var/log/httpd/zsell_ai-error_log"
    CustomLog "/opt/homebrew/var/log/httpd/zsell_ai-access_log" common
</VirtualHost>
```

This configuration points to `/Users/wernerheigl/Sites/zsell-ai-wordpress` as your WordPress installation directory. Make sure that this directory exists and contains your WordPress files.

If you need to add additional configuration options to your virtual host, you can include directory-specific settings:

```
<VirtualHost *:8080>
    ServerAdmin webmaster@zsell.ai
    ServerName zsell.ai.local
    DocumentRoot "/Users/wernerheigl/Sites/zsell-ai-wordpress"
    ErrorLog "/opt/homebrew/var/log/httpd/zsell_ai-error_log"
    CustomLog "/opt/homebrew/var/log/httpd/zsell_ai-access_log" common
    
    <Directory "/Users/wernerheigl/Sites/zsell-ai-wordpress">
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

### Step 6: Configure Hosts File for Local Domain

For your `zsell.ai.local` domain to work properly, you need to make sure your system knows how to resolve this domain:

```bash
# Edit the hosts file
sudo nano /etc/hosts
```

Make sure you have this line in the file (if it's not there, add it):
```
127.0.0.1   zsell.ai.local
```

Save and close the file.

**Important**: Since Apache is running on port 8080, you'll need to include the port in your URLs:
```
http://zsell.ai.local:8080
http://zsell.ai.local:8080/wp-login.php
```

```bash
# Get Homebrew prefix
BREW_PREFIX=$(brew --prefix)

# Create log directories for Apache
mkdir -p $BREW_PREFIX/var/log/httpd
```

### Step 7: Create WordPress Directory

We'll be using the ~/Sites directory for our WordPress installations:

```bash
# Create the Sites directory if it doesn't exist
mkdir -p ~/Sites

# Create the WordPress directory within Sites
mkdir -p ~/Sites/zsell-ai-wordpress

# Set proper permissions for all subdirectories
find ~/Sites -type d -exec chmod 755 {} \;
```

If you're still having issues with "Forbidden" errors when accessing subdirectories, try these additional steps:

```bash
# Make sure your home directory has the execute permission for others
chmod o+x ~

# Make sure all directories in the path have the proper permissions
chmod 755 ~/Sites
chmod -R 755 ~/Sites/zsell-ai-wordpress
```

### Step 8: Restart Apache

```bash
# Get Homebrew prefix
BREW_PREFIX=$(brew --prefix)

# Check Apache configuration
$BREW_PREFIX/bin/httpd -t

# Check Apache virtual hosts configuration
$BREW_PREFIX/bin/httpd -S

# Restart Apache to apply changes
brew services restart httpd
```

## WordPress Installation

### Step 1: Create Database for WordPress

```bash
# Log in to MySQL
mysql -u root -p

# Enter the password you set during mysql_secure_installation
```

Once logged in to MySQL, run these commands:
```sql
CREATE DATABASE wordpress;
CREATE USER 'wordpressuser'@'localhost' IDENTIFIED BY 'your_strong_password';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpressuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Replace `your_strong_password` with a secure password.

### Step 2: Download and Configure WordPress

If you're using Laravel Herd or another PHP environment manager, you'll need to modify the PHP configuration for WP-CLI:

```bash
# For Laravel Herd, edit the php.ini file
# The location may vary but is often at:
nano ~/.config/herd/config/php/8.x/php.ini
# or
nano ~/.herd/config/php/8.x/php.ini
```

Find the `memory_limit` setting and increase it:
```
memory_limit = 512M
```

Save the file and restart Herd if necessary.

If you're using the standard Homebrew PHP:
```bash
# Create or update your WP-CLI configuration file
mkdir -p ~/.wp-cli
echo "memory_limit = 512M" > ~/.wp-cli/config.yml
```

Now proceed with downloading and configuring WordPress:

```bash
# Navigate to your WordPress directory
cd ~/Sites/zsell.ai  # Use your actual WordPress directory path

# Download WordPress core files
wp core download

# If you still have memory issues, run WP-CLI with increased memory directly
php -d memory_limit=512M $(which wp) core download

# Create WordPress configuration file
wp config create --dbname=wordpress --dbuser=wordpressuser --dbpass=your_strong_password --dbhost=localhost

# Install WordPress
wp core install --url=zsell.ai.local:8080 --title="ZSell" --admin_user=admin --admin_password=your_admin_password --admin_email=your@email.com
```

Replace `your_strong_password`, `your_admin_password`, and `your@email.com` with your own values. Also, make sure to use the correct database name that you've created.

Note that we're using `zsell.ai.local:8080` as the URL since Apache is running on port 8080. If you didn't include the port in the `--url` parameter during installation, you'll need to update the site URL and home URL:

```bash
# Update the WordPress site URL to include the port
wp option update siteurl 'http://zsell.ai.local:8080'
wp option update home 'http://zsell.ai.local:8080'
```

You can access your WordPress installation using:
- http://zsell.ai.local:8080/

## Git Version Control Setup

### Step 1: Initialize Git Repository

```bash
# Navigate to your WordPress directory
cd ~/Sites/your_project_name

# Initialize a Git repository
git init
```

## Git Strategy for WordPress Block Editor Site

For a site using WordPress 6.9's native Block Editor and Site Editor with the TwentyTwentyFive theme, the Git strategy is straightforward. Since Site Editor customizations (global styles, template modifications, navigation menus) are stored in the database rather than in theme files, version control focuses primarily on:

1. **Custom plugins** you develop
2. **Configuration files** (excluding sensitive data)
3. **Any custom functionality plugins**

### What to Track in Version Control

1. **Custom Functionality Plugins**:
   ```
   /wp-content/plugins/zsell-custom-functionality/
   ```
   Track any custom plugins you create for site-specific functionality.

2. **Must-Use Plugins** (if any):
   ```
   /wp-content/mu-plugins/
   ```

3. **Custom CSS/JS Files** (if stored outside the database):
   ```
   /wp-content/themes/twentytwentyfive-child/assets/  # If using a child theme
   ```

### What to Exclude from Version Control

1. **WordPress Core Files**: Standard WordPress files that can be reinstalled.

2. **User Uploads**:
   ```
   /wp-content/uploads/
   ```
   Media files should be synced separately, not through Git.

3. **Sensitive Configuration**:
   ```
   wp-config.php
   .htaccess
   ```

4. **Themes** (including TwentyTwentyFive):
   ```
   /wp-content/themes/
   ```
   Stock themes can be reinstalled. Site Editor customizations are in the database.

5. **Standard Plugins**:
   ```
   /wp-content/plugins/
   ```
   Plugins from the WordPress repository can be reinstalled via WP-CLI.

6. **Development Files and Caches**:
   ```
   node_modules/
   .DS_Store
   wp-content/cache/
   ```

### Recommended `.gitignore` for Block Editor Site

```gitignore
# WordPress Core
wp-admin/
wp-includes/
wp-content/index.php
wp-content/languages/
wp-content/plugins/index.php
wp-content/themes/index.php
index.php
license.txt
readme.html
wp-*.php
!wp-config-sample.php
xmlrpc.php

# Configuration (with sensitive data)
wp-config.php
.env
.htaccess

# User uploads
wp-content/uploads/

# Cache and temporary files
wp-content/cache/
wp-content/upgrade/
wp-content/backup-db/
wp-content/backups/
wp-content/blogs.dir/
wp-content/wp-cache-config.php
*.log

# All themes (Site Editor changes are stored in database)
wp-content/themes/

# All plugins except custom ones
wp-content/plugins/*
!wp-content/plugins/zsell-custom-functionality/
# Add other custom plugins as needed

# OS and editor files
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db
.idea/
.vscode/
*.swp
*.swo

# Database exports
*.sql
*.gz
```

### Key Principle: Database is Primary

With WordPress 6.9 and the Site Editor, your design customizations live in the database:
- **Global Styles** (colors, typography, spacing)
- **Template modifications**
- **Template parts** (headers, footers)
- **Navigation menus**
- **Block patterns** (when customized)

This means **database synchronization is more critical than Git** for design continuity across environments. See the [Database Synchronization](#database-synchronization) section for scripts to manage this.

## Content Workflow with Block Editor

For a site where team members create content using the Block Editor on the staging environment, the workflow centers on database synchronization rather than file tracking.

### Content Creation Workflow

#### 1. Content Creation on Staging

1. **Content creators work directly on staging**:
   - Create pages and posts using the Block Editor
   - Use TwentyTwentyFive's built-in block patterns for consistent layouts
   - Customize global styles via Appearance > Editor > Styles

2. **Design changes via Site Editor**:
   - Modify templates and template parts
   - Adjust global color palette and typography
   - All changes automatically saved to database

#### 2. Review and Approval

1. Management reviews content on staging
2. Once approved, content is ready for production deployment

#### 3. Production Deployment

Since all content and design changes are in the database:

```bash
# Export staging database
ssh username@your_domain.com "cd /path/to/staging/public_html && wp db export staging-db-export.sql"

# Create production backup first
ssh username@your_domain.com "cd /path/to/production/public_html && wp db export production-backup-$(date +%Y%m%d).sql"

# Transfer and import to production
ssh username@your_domain.com "cp /path/to/staging/public_html/staging-db-export.sql /path/to/production/public_html/"
ssh username@your_domain.com "cd /path/to/production/public_html && wp db import staging-db-export.sql"

# Update URLs
ssh username@your_domain.com "cd /path/to/production/public_html && wp search-replace 'staging.your_domain.com' 'your_domain.com' --all-tables"

# Cleanup
ssh username@your_domain.com "rm /path/to/staging/public_html/staging-db-export.sql /path/to/production/public_html/staging-db-export.sql"
```

### Selective Content Migration

For migrating specific pages rather than the entire database:

```bash
# On staging, export specific content
wp export --post_type=page --post__in=123,456 --filename=landing-pages.xml

# Transfer the file to production
scp landing-pages.xml user@production-server:/tmp/

# On production, import the content
wp import /tmp/landing-pages.xml --authors=create
```

### Design System Consistency

The Site Editor's global styles ensure design consistency:

1. **Colors**: Define your brand palette once in Styles > Colors
2. **Typography**: Set font families and sizes in Styles > Typography
3. **Spacing**: Configure default spacing in Styles > Layout

These settings propagate to all blocks automatically, ensuring the ZSell design system is applied consistently.

### Training Guidelines for Content Creators

1. **Block Editor Best Practices**:
   - Use the built-in TwentyTwentyFive patterns for consistent layouts
   - Apply global styles rather than inline styling
   - Use reusable blocks for repeated content elements

2. **Workflow Guidelines**:
   - Always work on staging first
   - Request review before production deployment
   - Test responsive behavior using the Block Editor's preview modes

### Step 3: Add wp-config-sample.php to Repository

Since we're excluding wp-config.php, we should include a sample configuration file:

```bash
# Copy the sample configuration file
cp wp-config.php wp-config-sample.php

# Remove any sensitive information from the sample
nano wp-config-sample.php
```

Replace database credentials with placeholders in the sample file.

### Step 4: Make Initial Commit

```bash
# Add all files to staging (except those in .gitignore)
git add .

# Make initial commit
git commit -m "Initial WordPress setup"
```

## Remote Repository Setup

You have two options for setting up your remote repository: GitHub or GoDaddy.

### Option 1: GitHub Repository Setup

1. Create a new repository on GitHub through the web interface
2. Connect your local repository to GitHub:

```bash
# Add GitHub as remote origin
git remote add origin https://github.com/yourusername/your-repository-name.git

# Rename main branch if needed (modern GitHub repositories use 'main' instead of 'master')
git branch -M main

# Push your local repository to GitHub
git push -u origin main
```

### Option 2: GoDaddy Repository Setup

If GoDaddy provides Git repository hosting:

```bash
# Add GoDaddy as remote origin
git remote add origin ssh://your_godaddy_username@your_domain.com/path/to/git/repository

# Push your local repository to GoDaddy
git push -u origin main
```

## Staging Environment Configuration

### Step 1: Create Subdomain on GoDaddy

1. Log in to your GoDaddy account
2. Navigate to the Domain Management section
3. Find your domain and select "DNS Management"
4. Add a new "A" record with:
   - Host: staging
   - Points to: Your server's IP address
   - TTL: 1 hour

### Step 2: Set Up Web Server for Staging

SSH into your hosting server:

```bash
ssh username@your_domain.com
```

Create a directory for the staging site:

```bash
# Create staging directory
mkdir -p /path/to/staging/public_html

# Set proper permissions
chmod 755 /path/to/staging/public_html
```

Configure the web server (Apache) for the staging subdomain:

```bash
# For Apache, create a virtual host configuration
sudo nano /etc/apache2/sites-available/staging.your_domain.com.conf
```

Add this configuration:

```
<VirtualHost *:80>
    ServerName staging.your_domain.com
    DocumentRoot /path/to/staging/public_html
    
    <Directory /path/to/staging/public_html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    
    ErrorLog ${APACHE_LOG_DIR}/staging.your_domain.com_error.log
    CustomLog ${APACHE_LOG_DIR}/staging.your_domain.com_access.log combined
</VirtualHost>
```

Enable the site and restart Apache:

```bash
sudo a2ensite staging.your_domain.com.conf
sudo systemctl restart apache2
```

### Step 3: Set Up Git Deployment for Staging

On your staging server:

```bash
# Initialize a bare Git repository
mkdir -p ~/git-repos/staging.git
cd ~/git-repos/staging.git
git init --bare
```

Create a post-receive hook:

```bash
nano ~/git-repos/staging.git/hooks/post-receive
```

Add the following content:

```bash
#!/bin/bash
echo "Deploying to staging environment..."
GIT_WORK_TREE=/path/to/staging/public_html git checkout -f
cd /path/to/staging/public_html

# Run any necessary commands after deployment
# For example, updating WordPress URLs if needed
echo "Deployment completed!"
```

Make the hook executable:

```bash
chmod +x ~/git-repos/staging.git/hooks/post-receive
```

### Step 4: Add Staging Remote to Local Repository

On your local machine:

```bash
# Add staging remote
git remote add staging ssh://username@your_domain.com/~/git-repos/staging.git
```

## Production Environment Setup

### Step 1: Set Up Git Deployment for Production

Similar to staging, set up a bare Git repository on your production server:

```bash
# SSH into your production server
ssh username@your_domain.com

# Create a bare Git repository
mkdir -p ~/git-repos/production.git
cd ~/git-repos/production.git
git init --bare
```

Create a post-receive hook:

```bash
nano ~/git-repos/production.git/hooks/post-receive
```

Add the following content:

```bash
#!/bin/bash
echo "Deploying to production environment..."
GIT_WORK_TREE=/path/to/production/public_html git checkout -f
cd /path/to/production/public_html

# Run any necessary commands after deployment
echo "Production deployment completed!"
```

Make the hook executable:

```bash
chmod +x ~/git-repos/production.git/hooks/post-receive
```

### Step 2: Add Production Remote to Local Repository

On your local machine:

```bash
# Add production remote
git remote add production ssh://username@your_domain.com/~/git-repos/production.git
```

## Database Synchronization

Database synchronization is critical for WordPress Block Editor sites since all Site Editor customizations, content, and design settings are stored in the database.

### Step 1: Install WP-CLI on Staging and Production

If not already installed, add WP-CLI to your staging and production environments:

```bash
# SSH into your server (staging or production)
ssh username@your_domain.com

# Download WP-CLI
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar

# Make it executable
chmod +x wp-cli.phar

# Move it to a directory in your PATH
sudo mv wp-cli.phar /usr/local/bin/wp
```

### Step 2: Create Database Synchronization Scripts

#### Script to Push Local Database to Staging

Create this script on your local machine:

```bash
# Create a script file
nano db-push-to-staging.sh
```

Add the following content:

```bash
#!/bin/bash
echo "Exporting local database..."
wp db export wordpress-db-export.sql

echo "Transferring database to staging..."
scp wordpress-db-export.sql username@your_domain.com:/path/to/staging/public_html/

echo "Importing database on staging..."
ssh username@your_domain.com "cd /path/to/staging/public_html && wp db import wordpress-db-export.sql && rm wordpress-db-export.sql"

echo "Updating URLs in staging database..."
ssh username@your_domain.com "cd /path/to/staging/public_html && wp search-replace 'zsell.ai.local' 'staging.your_domain.com' --all-tables"

echo "Cleaning up..."
rm wordpress-db-export.sql

echo "Database push to staging completed!"
```

#### Script to Push Staging Database to Production

Create this script on your local machine:

```bash
# Create a script file
nano db-push-to-production.sh
```

Add the following content:

```bash
#!/bin/bash
echo "Creating backup of production database..."
ssh username@your_domain.com "cd /path/to/production/public_html && wp db export production-backup-$(date +%Y%m%d).sql"

echo "Exporting staging database..."
ssh username@your_domain.com "cd /path/to/staging/public_html && wp db export staging-db-export.sql"

echo "Transferring staging database to production..."
ssh username@your_domain.com "cp /path/to/staging/public_html/staging-db-export.sql /path/to/production/public_html/"

echo "Importing staging database to production..."
ssh username@your_domain.com "cd /path/to/production/public_html && wp db import staging-db-export.sql && rm staging-db-export.sql"

echo "Updating URLs in production database..."
ssh username@your_domain.com "cd /path/to/production/public_html && wp search-replace 'staging.your_domain.com' 'your_domain.com' --all-tables"

echo "Cleaning up..."
ssh username@your_domain.com "cd /path/to/staging/public_html && rm staging-db-export.sql"

echo "Database push to production completed!"
```

#### Script to Pull Production Database to Local

Create this script on your local machine:

```bash
# Create a script file
nano db-pull-from-production.sh
```

Add the following content:

```bash
#!/bin/bash
echo "Exporting production database..."
ssh username@your_domain.com "cd /path/to/production/public_html && wp db export production-db-export.sql"

echo "Transferring production database to local..."
scp username@your_domain.com:/path/to/production/public_html/production-db-export.sql ./

echo "Importing production database locally..."
wp db import production-db-export.sql

echo "Updating URLs in local database..."
wp search-replace 'your_domain.com' 'zsell.ai.local' --all-tables

echo "Cleaning up..."
rm production-db-export.sql
ssh username@your_domain.com "cd /path/to/production/public_html && rm production-db-export.sql"

echo "Database pull from production completed!"
```

Make all scripts executable:

```bash
chmod +x db-push-to-staging.sh db-push-to-production.sh db-pull-from-production.sh
```

## Site Editor and Theme Configuration

WordPress 6.9 with the TwentyTwentyFive theme provides a complete site-building experience through the native Block Editor and Site Editor. No additional page builder plugins are required.

### Step 1: Activate TwentyTwentyFive Theme

```bash
# Navigate to your WordPress directory
cd ~/Sites/your_project_name

# Ensure TwentyTwentyFive is installed (comes bundled with WordPress 6.9)
wp theme install twentytwentyfive --activate

# Verify activation
wp theme status twentytwentyfive
```

### Step 2: Access the Site Editor

The Site Editor is accessed via:
- **WordPress Admin** > **Appearance** > **Editor**

Or directly at:
```
http://zsell.ai.local:8080/wp-admin/site-editor.php
```

### Step 3: Configure Global Styles

In the Site Editor, configure your brand's design system:

1. **Open Styles Panel**: Click the Styles icon (half-filled circle) in the top-right
2. **Colors**: Set your brand palette
   - Navigate to Styles > Colors
   - Define primary, secondary, and accent colors matching ZSell brand guidelines
3. **Typography**: Configure fonts
   - Navigate to Styles > Typography
   - Set heading and body font families, sizes, and weights
4. **Layout**: Set spacing defaults
   - Navigate to Styles > Layout
   - Configure content width and padding

### Step 4: Customize Templates

TwentyTwentyFive includes a variety of templates you can customize:

1. **Templates**: Modify page layouts
   - Home, Single Post, Page, Archive, 404, etc.
2. **Template Parts**: Edit reusable sections
   - Header, Footer, Sidebar

Access via Site Editor > Templates or Template Parts.

### Step 5: Use Block Patterns

TwentyTwentyFive provides numerous block patterns for common layouts:

1. In the Block Editor, click the **+** button to add a block
2. Select the **Patterns** tab
3. Browse categories: Hero, Features, Testimonials, CTAs, etc.
4. Insert and customize as needed

### Key Benefits of Native Block Editor

1. **No Plugin Dependencies**: Everything is built into WordPress core
2. **Performance**: No additional JavaScript frameworks to load
3. **Future-Proof**: Follows WordPress's development roadmap
4. **Database-Stored Customizations**: Theme updates won't overwrite your changes
5. **Pattern Library**: Extensive pre-built layouts in TwentyTwentyFive

## Development and Deployment Workflow

### Complete Development Workflow

#### Step 1: Starting a New Feature

```bash
# Pull the latest changes from the main repository
git pull origin main

# Create a new branch for your feature
git checkout -b feature/new-feature-name

# Update your local database with the latest from production (optional)
./db-pull-from-production.sh
```

#### Step 2: Making Changes

1. Make your changes using WordPress admin and the Block Editor
2. Use the Site Editor for template and global style modifications
3. Test your changes locally at http://zsell.ai.local:8080

#### Step 3: Committing Changes

For code changes (custom plugins, configuration):

```bash
# Add your changes to Git
git add .

# Commit your changes
git commit -m "Description of your changes"

# Push your feature branch to the remote repository
git push origin feature/new-feature-name
```

**Note**: Site Editor changes (templates, global styles) are stored in the database. Commit any related documentation or configuration files, but the actual design changes will be migrated via database sync.

#### Step 4: Code Review (if collaborating with others)

1. Create a pull request on GitHub
2. Wait for code review and approval
3. Merge to main branch

#### Step 5: Deploy to Staging

```bash
# Switch to main branch
git checkout main

# Pull latest changes if using GitHub and pull requests
git pull origin main

# Push code to staging
git push staging main

# Sync database to staging (includes Site Editor changes)
./db-push-to-staging.sh
```

#### Step 6: Testing on Staging

1. Test thoroughly on https://staging.your_domain.com
2. Verify Site Editor customizations rendered correctly
3. Check responsive behavior
4. Make any necessary adjustments on staging
5. If staging changes need to come back to local, pull the staging database

#### Step 7: Deploy to Production

```bash
# After thorough testing on staging
git push production main

# Sync database to production
./db-push-to-production.sh
```

### Important: Database is the Source of Truth

With the Block Editor approach:
- **Git** tracks code (custom plugins, scripts, configuration)
- **Database** contains design (Site Editor changes, content, menus)

Always sync the database when deploying design or content changes.

## Collaboration Guide

### Setting Up for a New Developer

1. New developer clones the repository:
   ```bash
   git clone https://github.com/yourusername/your-repository-name.git
   cd your-repository-name
   ```

2. New developer sets up local environment:
   - Follow the steps in Local Development Environment Setup
   - Create a wp-config.php file based on wp-config-sample.php
   - Create and configure local database
   - Install WordPress core and TwentyTwentyFive theme:
     ```bash
     wp core download
     wp theme activate twentytwentyfive
     ```
   - Import the latest database:
     ```bash
     ./db-pull-from-production.sh
     ```

3. New developer adds remotes:
   ```bash
   git remote add staging ssh://username@your_domain.com/~/git-repos/staging.git
   git remote add production ssh://username@your_domain.com/~/git-repos/production.git
   ```

### Working with Feature Branches

1. Always create feature branches for new work:
   ```bash
   git checkout -b feature/descriptive-name
   ```

2. Regularly pull changes from main:
   ```bash
   git checkout main
   git pull origin main
   git checkout feature/descriptive-name
   git rebase main
   ```

3. Push feature branches for collaboration:
   ```bash
   git push origin feature/descriptive-name
   ```

## Troubleshooting

### Common Local Development Issues

#### Apache Not Starting

```bash
# Get Homebrew prefix
BREW_PREFIX=$(brew --prefix)

# Check Apache error logs
cat $BREW_PREFIX/var/log/httpd/error_log

# Verify Apache configuration
$BREW_PREFIX/bin/httpd -t
```

#### Multiple Apache Instances Running

If you discover both the system Apache and Homebrew Apache running:

```bash
# Check running Apache processes
ps aux | grep httpd

# Stop the system Apache
sudo killall httpd

# Disable system Apache from starting at boot
sudo launchctl unload -w /System/Library/LaunchDaemons/org.apache.httpd.plist

# Restart Homebrew Apache
brew services restart httpd
```

#### Database Connection Issues

1. Verify database credentials in wp-config.php
2. Ensure MySQL is running:
   ```bash
   brew services list
   # If MySQL is not running
   brew services start mysql
   ```
3. Test database connection:
   ```bash
   mysql -u wordpressuser -p
   ```

#### WP-CLI Memory Issues

If you encounter "Fatal error: Allowed memory size exhausted" when using WP-CLI:

##### For Laravel Herd:
```bash
# Edit Laravel Herd's PHP configuration
# Find the correct PHP version you're using (e.g., 8.2)
nano ~/.config/herd/config/php/8.2/php.ini
# or sometimes
nano ~/.herd/config/php/8.2/php.ini

# Find the memory_limit line and change it to:
memory_limit = 512M

# Restart Herd to apply changes
```

##### For Homebrew PHP:
```bash
# Create a WP-CLI config file with increased memory limit
mkdir -p ~/.wp-cli
echo "memory_limit = 512M" > ~/.wp-cli/config.yml

# Alternative: Run commands with increased memory limit for a single command
php -d memory_limit=512M $(which wp) core download

# Alternative: Set a global PHP memory limit
BREW_PREFIX=$(brew --prefix)
echo "memory_limit = 512M" > $BREW_PREFIX/etc/php/*/conf.d/99-memory-limit.ini
```

You can verify which PHP configuration file is being used with:
```bash
php --ini | grep "Loaded Configuration File"
```

#### Permission Issues

If you see "403 Forbidden" errors when accessing subdirectories:

```bash
# Fix WordPress directory permissions
chmod o+x ~  # Allow others to execute in your home directory
chmod 755 ~  # Alternative approach if the above doesn't work
chmod 755 ~/Sites
chmod -R 755 ~/Sites/wordpress

# If you're still having issues, check the permissions of each directory in the path
ls -la ~
ls -la ~/Sites
ls -la ~/Sites/wordpress
```

Also check the Apache error log for specific permission issues:

```bash
BREW_PREFIX=$(brew --prefix)
cat $BREW_PREFIX/var/log/httpd/error_log
```

Possible errors and solutions:

1. **"client denied by server configuration"**: Check your Apache configuration to ensure subdirectories are allowed.
2. **"Permission denied"**: Fix file/directory permissions.
3. **"failed to open dir: No such file or directory"**: Check if the path exists and is accessible.

You can also add a `.htaccess` file to your Sites directory:

```bash
echo "Require all granted" > ~/Sites/.htaccess
echo "Require all granted" > ~/Sites/wordpress/.htaccess
```

### Common Deployment Issues

#### Git Push Errors

```bash
# If you get "Permission denied" errors
# Verify SSH access to your server
ssh username@your_domain.com

# Test Git connection
ssh -T username@your_domain.com
```

#### Hook Script Errors

1. Check if the post-receive hook is executable:
   ```bash
   ssh username@your_domain.com "ls -la ~/git-repos/staging.git/hooks/post-receive"
   ```
2. Verify the hook script path:
   ```bash
   ssh username@your_domain.com "cat ~/git-repos/staging.git/hooks/post-receive"
   ```

#### URL Issues After Database Sync

If site URLs are incorrect after database sync:

```bash
# For staging
ssh username@your_domain.com "cd /path/to/staging/public_html && wp search-replace 'zsell.ai.local' 'staging.your_domain.com' --all-tables"

# For production
ssh username@your_domain.com "cd /path/to/production/public_html && wp search-replace 'staging.your_domain.com' 'your_domain.com' --all-tables"
```

### General WordPress Issues

#### White Screen of Death

1. Enable WordPress debugging:
   Edit wp-config.php and add:
   ```php
   define('WP_DEBUG', true);
   define('WP_DEBUG_LOG', true);
   define('WP_DEBUG_DISPLAY', false);
   ```
2. Check the debug.log file:
   ```bash
   cat wp-content/debug.log
   ```

#### Plugin or Theme Conflicts

1. Deactivate all plugins:
   ```bash
   wp plugin deactivate --all
   ```
2. Switch to TwentyTwentyFive if using a different theme:
   ```bash
   wp theme activate twentytwentyfive
   ```
3. Reactivate plugins one by one to identify conflicts.

#### Site Editor Not Loading

If the Site Editor fails to load:

1. Clear browser cache
2. Check for JavaScript errors in browser console
3. Verify WordPress and theme are up to date:
   ```bash
   wp core update
   wp theme update twentytwentyfive
   ```
4. Check for conflicting plugins:
   ```bash
   wp plugin deactivate --all
   ```

### Configuring Apache to Use Standard Port 80

For the best compatibility with WordPress (especially for production deployments), it's recommended to use the standard HTTP port 80. Let's explore the options for macOS:

#### Option 1: Port Forwarding from 80 to 8080 (Recommended for macOS)

On macOS, the simplest approach is to keep Apache running on port 8080 (which doesn't require special privileges) and set up port forwarding from port 80:

1. **Make sure Apache is configured for port 8080**:
   ```bash
   # Get Homebrew prefix
   BREW_PREFIX=$(brew --prefix)
   
   # Edit the main Apache configuration file
   nano $BREW_PREFIX/etc/httpd/httpd.conf
   ```

   Verify this line is present:
   ```
   Listen 8080
   ```

2. **Set up port forwarding using pfctl** (macOS Packet Filter):
   ```bash
   # Create a temporary pf configuration file
   echo "rdr pass inet proto tcp from any to any port 80 -> 127.0.0.1 port 8080" > /tmp/pf.conf
   
   # Enable the port forwarding rule
   sudo pfctl -f /tmp/pf.conf -e
   ```

   If you get an error about pf being disabled, enable it first:
   ```bash
   # Enable pf
   sudo pfctl -e
   
   # Then apply the rules
   sudo pfctl -f /tmp/pf.conf
   ```

3. **Make port forwarding persistent across reboots** (optional):
   ```bash
   # Create a permanent pf anchors file
   sudo nano /etc/pf.anchors/com.user.port-forwarding
   ```
   
   Add this content:
   ```
   rdr pass inet proto tcp from any to any port 80 -> 127.0.0.1 port 8080
   ```
   
   Then edit the main pf configuration:
   ```bash
   sudo nano /etc/pf.conf
   ```
   
   Add this line at the end:
   ```
   anchor "com.user.port-forwarding"
   ```

With port forwarding set up, verify that it's working:
```bash
# Test if port 80 is being forwarded
curl -I http://zsell.ai.local
```

You should see a successful HTTP response without specifying port 8080.

#### Option 2: Run Apache Directly on Port 80 (Requires Root)

If you prefer to run Apache directly on port 80:

1. **Change Apache to listen on port 80**:
   ```bash
   # Edit the main Apache configuration file
   BREW_PREFIX=$(brew --prefix)
   nano $BREW_PREFIX/etc/httpd/httpd.conf
   ```

   Change:
   ```
   Listen 8080
   ```
   
   To:
   ```
   Listen 80
   ```

2. **Update your virtual host configuration**:
   ```bash
   # Edit your virtual host configuration
   nano $BREW_PREFIX/etc/httpd/vhosts/zsell.ai.conf
   ```

   Change:
   ```
   <VirtualHost *:8080>
   ```
   
   To:
   ```
   <VirtualHost *:80>
   ```

3. **Start Apache with sudo** (required for privileged ports):
   ```bash
   # Stop the regular Apache service
   brew services stop httpd
   
   # Start Apache with sudo
   sudo $BREW_PREFIX/bin/httpd -k start
   ```

#### Update WordPress Site URL Configuration

After setting up either port forwarding or direct port 80 configuration, update your WordPress configuration to use the standard URL without a port:

```bash
# Navigate to your WordPress directory
cd ~/Sites/zsell.ai

# Update your site URL to not include a port
wp option update siteurl 'http://zsell.ai.local'
wp option update home 'http://zsell.ai.local'
```

Now you can access your WordPress site without specifying a port:
```
http://zsell.ai.local/
http://zsell.ai.local/wp-login.php
```

```bash
# Get Homebrew prefix
BREW_PREFIX=$(brew --prefix)

# Check Apache error logs
cat $BREW_PREFIX/var/log/httpd/error_log

# Verify Apache configuration
$BREW_PREFIX/bin/httpd -t
```

---

This guide provides a comprehensive workflow for WordPress development from local setup to production deployment using WordPress 6.9's native Block Editor and Site Editor with the TwentyTwentyFive theme. Remember to adapt paths, URLs, and credentials to match your specific environment.
