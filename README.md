# Arthub Ensor

## Project overview

Arthub Ensor is a fork of the [Arthub Flanders](https://github.com/vlaamsekunstcollectie/Arthub-Frontend/tree/iiif-testbed) platform, a customised [Project Blacklight](https://github.com/projectblacklight/blacklight) installation written in Ruby on Rails. It utilises an [Imagehub](https://github.com/Hero-Solutions/ImageHub/tree/ensor) installation to generate [IIIF Presentation API](https://iiif.io/api/presentation/2.1/) manifests.

The Imagehub provides a link between [ResourceSpace](https://www.resourcespace.com) and the [Datahub](https://github.com/thedatahub/Datahub). High-resolution images are uploaded in ResourceSpace and data from the Datahub is added to these resources through the use of an Imagehub command. A second command generates IIIF Presentation API manifests for each resource and a link to each manifest is added to the appropriate record inside the Datahub. Through the use of [Catmandu](https://librecat.org/Catmandu/) fixes, each Datahub record can be transformed into an Apache Solr document to be displayed on the Arthub with metadata about the work along with the appropriate high-resolution image inside a [IIIF viewer](https://iiif.io/apps-demos/#image-viewing-clients).

This project does not make use of the [IIIF Authentication API](https://iiif.io/api/auth/1.0/), as all images are publicly available. Anything related to authentication is therefore not included in this document.

## Requirements

The complete Arthub Ensor installation requires the following components:

* ImageMagick
* ExifTool
* SQLite3
* Perl >= 5.26
* CPANMinus
* libssl-dev
* libxslt1-dev
* PHP >= 7.2 with the following extensions (not all Arthub components may be compatible with PHP >= 7.3, so PHP 7.2 is recommended at the time of writing):
  * php-mbstring
  * php-mysql
  * php-simplexml
  * php-curl
  * php-gd
  * php-imagick
* Ruby >= 2.4, Bundler >= 1.16, Rails >= 5.1 and the SQLite3 gem (we recommend to install these through [rvm](https://rvm.io/rvm/install))
* MySQL or MariaDB database server
* Apache or NGinx webserver
* [Cantaloupe](https://cantaloupe-project.github.io/) >= 4.1
* [ResourceSpace](https://www.resourcespace.com/get) >= 9.1 with the [RS_ptif](https://github.com/kmska/RS_ptif) plugin installed, this may be installed on a different server

## Preparation

In order to add Datahub metadata to the appropriate resources, you first need to generate a CSV file that contains two columns: the image file name and the Datahub record ID that contains metadata about the work. A sample CSV file (for two images) may look like this:
```
filename;datahub_record_id
1857.jpg;oai:datahub.vlaamsekunstcollectie.be:kmska.be:1857
1858.001.jpeg;oai:datahub.vlaamsekunstcollectie.be:kmska.be:1858
```

This CSV will be required by the Imagehub later on. In this example, we will assume the filename is 'ensor.csv'.

Your ResourceSpace installation also requires a specific set of metadata fields. You can set this up by using the resourcespace_metadata_fields.sql file included in the [Imagehub](https://github.com/Hero-Solutions/Imagehub/tree/ensor) project, this will drop and recreate the resource_type_field table. You can do so by running the following command on the server where ResourceSpace is installed:
```
mysql -u resourcespace -presourcespace resourcespace < resourcespace_metadata_fields.sql
```

Certain metadata fields, most notably dropdown lists (for example Publisher and Cleared for usage) need to be prefilled with the necessary values before adding resources. This can be done either manually through the admin console of ResourceSpace or by using the resourcespace_node_values.sql included in the [Imagehub](https://github.com/Hero-Solutions/Imagehub/tree/ensor):
```
mysql -u resourcespace -presourcespace resourcespace < resourcespace_node_values.sql
```

The Imagehub itself also requires its very own database containing a table 'iiif_manifest' according to the following structure:
```
CREATE TABLE `iiif_manifest` (
  `id` int UNSIGNED NOT NULL AUTO_INCREMENT,
  `manifest_id` varchar(255) NOT NULL,
  `data` longtext NOT NULL,
  PRIMARY KEY (`id`)
);
```
A MySQL user is to be created with full access to this table. The username, password and database name can be freely chosen and will be configured in the .env file in the [Imagehub](https://github.com/Hero-Solutions/Imagehub/tree/ensor) repository (further in this document).

## Installation

### Starting Cantaloupe

Start the Cantaloupe image server by running the following command inside the Cantaloupe setup folder (adjust arguments where needed):
```
java -Dcantaloupe.config=/opt/cantaloupe/cantaloupe.properties -Xmx2g -jar /opt/cantaloupe/cantaloupe-4.1.5.war
```

### Imagehub installation

Clone the Imagehub repository and checkout the appropriate branch:
```
git clone https://github.com/Hero-Solutions/Imagehub.git Imagehub
cd Imagehub
git checkout ensor
```

Copy the .env.sample file to .env and the config/imagehub.yaml.sample file to config/imagehub.yaml. Edit any values according to your local setup in .env and config/imagehub.yaml. At the very least, the following values should be checked and edited where needed:

In .env:
* DATABASE_URL=mysql://(user):(pass)@(host):(port)/(db_name)

In config/imagehub.yaml:
* ResourceSpace API
    * resourcespace_api_url
    * resourcespace_api_username
    * resourcespace_api_key
* Datahub URL, API credentials and LIDO record id prefix
    * datahub_url
    * datahub_username
    * datahub_password
    * datahub_public_id
    * datahub_secret
    * datahub_record_id_prefix
* Cantaloupe URL
    * cantaloupe_url
* Service URL
    * service_url
* Public/private folder where to find the images through Cantaloupe
    * public_folder
    * private_folder


Then, install the Imagehub through [composer](https://getcomposer.org/):
```
cd Imagehub
composer install
```

### Arthub installation

Clone the Arthub, checkout the appropriate branch and install:
```
git clone https://github.com/VlaamseKunstcollectie/Arthub-Frontend.git arthub/
cd arthub
git checkout iiif-testbed
bundle install
rake db:migrate
rake solr:clean
cp -r solr-conf/blacklight-core/ solr/server/solr
rake solr:start
```

If rake solr:clean returns an error when attempting to download solr-x.x.x.zip.md5 (as it often does), you can manually download it, for example from https://archive.apache.org/dist/lucene/solr/7.3.1/solr-7.3.1.zip.md5 and copy the file to the 'tmp/' folder (relative to the project root), then run rake solr:clean again. If it still fails, it may be necessary to download solr-x.x.x.zip from the same website (https://archive.apache.org/dist/lucene/solr/7.3.1/) and put it in 'tmp/' as well.

Precompile all assets by running the following command on the command line:
```
RAILS_ENV=production bundle exec rake assets:precompile
```
This will create all the assets and store them in the 'public/assets' folder.

Set the environment variable RAILS_SERVE_STATIC_FILES in your .bash_profile file (or equivalent) accordingly:
```
export RAILS_SERVE_STATIC_FILES=true
```
Generate a secret key hash:
```
rails secret
```
This will spit out a long hash. Copy this hash and add it to your .bash_profile file (or equivalent) as an environment variable:
```
export SECRET_KEY_BASE="<HASH>"
```

You are now set to run the production server:
```
rails s -e production
```

### Datahub > Arthub pipeline preparation

Install the necessary Perl modules by running the following command as root:
```
cpanm Catmandu Catmandu::XML Catmandu::OAI Datahub::Factory Datahub::Factory::Cmd DBD::SQLite
```

Clone the Imagehub-Fixes repository:
```
git clone https://github.com/VlaamseKunstcollectie/Imagehub-Fixes.git Imagehub-Fixes
```

For this particular setup, we only want to retain those Datahub records that have a reference to a IIIF Manifest generated by the Imagehub. In order to do so, add the following lines of code in fixes/datahub-oai-to-blacklight-solr-nl.fix and fixes/datahub-oai-to-blacklight-solr-en.fix, before the line that contains "remove_field('id')" (almost at the end of the file):
```
# Retain only the relevant records for the Ensor Arthub

set_field('imagehub_ensor', 'false')

do list(path:'filteredAdministrativeMetadata.resourceWrap.resourceSet', var:r)

    if all_match('r.resourceID.source', 'ImagehubEnsor')
        set_field('imagehub_ensor', 'true')
    end

end

unless all_match('imagehub_ensor', 'true')
    reject()
end

remove_field('imagehub_ensor')
```

In fixes/datahub-oai-to-blacklight-solr-en.fix, also find and replace 'en' by 'nl' in the following three lines:
```
if all_match('dm.lang', 'en')
```
```
if all_match('am.lang', 'en')
```
```
if all_match('per.term.*.lang', 'en')
```
This will allow the pipeline to fetch Dutch metadata from the Datahub and put it in the English index in the Arthub.

### Webserver configuration

The necessary Apache or Nginx configuration should be written in order to access the Arthub, Imagehub and ResourceSpace installations (ideally through different subdomains).

A sample Nginx configuration file for the Arthub and Imagehub combined may look like this:
```
server {
    listen 80 default;
    listen [::]:80 default;

    root /opt/arthub;

    index index.php index.html index.htm;

    server_name arthubensor.vlaamsekunstcollectie.be;

    location / {
        proxy_pass         http://127.0.0.1:3000;
        proxy_redirect     off;

        proxy_set_header   Host             $host;
        proxy_set_header   X-Real-IP        $remote_addr;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
    }

    # Pass requests for the Imagehub to the appropriate PHP file
    location ~ /iiif/2/.*(manifest\.json|collection/top) {
        try_files $uri /imagehub.php$is_args$args;
    }

    # Pass all other requests for /iiif/.* to Cantaloupe
    location ~ ^/iiif/2/[0-9]+.*$ {
      proxy_pass         http://127.0.0.1:8182;
      proxy_redirect     off;

      proxy_set_header   Host             $host;
      proxy_set_header   X-Real-IP        $remote_addr;
      proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
    }

    location ~ ^/imagehub\.php(/|$) {
        alias /opt/imagehub/public;

        fastcgi_pass unix:/run/php/php7.2-fpm.sock;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include         fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;
        internal;
    }
}

```

A sample Nginx configuration file for ResourceSpace may look like this (note the client_max_body_size in order to upload large image files):
```
server {
    listen 80;
    listen [::]:80;

    root /opt/resourcespace;

    index index.php;

    server_name ingest.arthubensor.vlaamsekunstcollectie.be;

    location ~* \.php$ {
        fastcgi_pass unix:/run/php/php7.2-fpm.sock;
        include         fastcgi_params;
        fastcgi_param   SCRIPT_FILENAME    $document_root$fastcgi_script_name;
        fastcgi_param   SCRIPT_NAME        $fastcgi_script_name;
    }

    client_max_body_size 50000M;
}
```

### RS_ptif configuration

Once the Imagehub is correctly installed, it is important to add the appropriate config values for the RS_ptif ResourceSpace plugin. Since this installation does not make use of authentication or a remote standalone image viewer, certain values can be left empty. In this instance, the following lines may be appended at the end of the ResourceSpace configuration file (includes/config.php):
```
# RS_ptif configuration

# This must be set to NULL in order to fix a bug within ResourceSpace
# where resource files are not properly deleted if this value is set to anything other than NULL.
# This bug resides in include/resource_functions.php:2015.
$resource_deletion_state = NULL;


# You can use either $iiif_imagehub_commands if the Imagehub is locally installed
# or $iiif_imagehub_curl_calls if the Imagehub is installed remotely

# Commands to be locally executed after an image is uploaded.
# {ref} will be automatically replaced by the corresponding resource ID by the plugin.
$iiif_imagehub_commands = array(
);

# cURL calls to be executed after an image is uploaded.
# {ref} will be automatically replaced by the corresponding resource ID by the plugin.
$iiif_imagehub_curl_calls = array(
);

# The URL of a IIIF manifest generated by one of the above commands.
# {ref} will be automatically replaced by the corresponding resource ID by the plugin.
$iiif_imagehub_manifest_url = '';

# Clickable URLs to be shown above the preview image in ResourceSpace.
# {manifest_url} will be automatically replaced by the manifest URL as defined in the line above by the plugin.
# We need to pass through the Imagehub authenticator first (which will redirect to the viewer through the 'url' GET parameter) when m$
$iiif_imagehub_viewers = array(
);

# Name of the folder where the ptif files are stored (relative to the filestore/ directory).
# Must contain a leading and trailing slash.
$iiif_ptif_filestore = '/iiif_ptif/';

# Key (shorthand name of the metadata field) and value to determine which images are cleared for public
# Key can be set to NULL if all images should be private
$iiif_ptif_public_key = 'clearedforusage';
$iiif_ptif_public_value = 'Public use';

# Folder to store private images, trailing slash is important
$iiif_ptif_private_folder = 'private/';

# Folder to store public images, trailing slash is important
# Can be set to NULL if all images should be private
$iiif_ptif_public_folder = 'public/';

# CLI Commands to perform image conversion to PTIF.
# 'extensions' defines a list of file extensions and the command that should be used to convert images with these extensions to PTIF.
# 'command' should probably be 'vips im_vips2tiff' or 'convert', but accepts any installed command for image conversion (can be the full path to an executable).
# 'arguments' defines extra command line arguments for the conversion command.
# 'dest_prefix' will be prefixed to the destination path, necessary for convert.
# 'dest_postfix' will be postfixed to the destination path, necessary for vips.
$iiif_ptif_commands = array(
    # vips cannot properly handle psb, so we need to use convert instead.
    array(
        'extensions'   => array('jpg', 'jpeg', 'psb'),
        'command'      => 'convert',
        'arguments'    => '-define tiff:tile-geometry=256x256 -compress jpeg -quality 40 -depth 8',
        'dest_prefix'  => 'ptif:',
        'dest_postfix' => ''
    ),
    # define catchall command for all other extensions with '*'
    # vips is generally faster and consumes fewer resources than convert, so use wherever possible.
    array(
        'extensions'   => array('*'),
        'command'      => 'vips im_vips2tiff',
        'arguments'    => '',
        'dest_prefix'  => '',
        'dest_postfix' => ':jpeg:40,tile:256x256,pyramid'
    )
);
```

## Usage

### Uploading images

Once all is set up correctly, you can start uploading images in bulk to ResourceSpace, making sure the following metadata fields are correctly set upon uploading:
* Publisher
* Cleared for usage (expand the entire tree and check everything including 'public use')
* Recommended image for publication (checked)

### Adding Datahub metadata to ResourceSpace

Once the images have been uploaded, execute the following command in the Imagehub installation folder (where 'ensor.csv' is the CSV file we generated at the start of the preparation):
```
php bin/console app:csv-datahub-to-resourcespace ensor.csv
```

This will match each row in the CSV with each resource in ResourceSpace by filename, fetch the appropriate data from the Datahub and add this data to the resource.

### Generating and storing IIIF manifests

Once the appropriate metadata is added to each resource, execute the following command in order to generate IIIF Presentation API manifests for each resource:
```
php bin/console app:generate-iiif-manifests
```
All manifests will be stored in a MySQL database and a link to each manifest will be added to the appropriate Datahub records.

The Imagehub also provides the appropriate logging. Logs are stored in var/logs/.

### Adding records to the Arthub

Last of all, run the following two commands inside the Imagehub-Fixes folder:
```
bash arthub-index.sh -e http://datahub.vlaamsekunstcollectie.be/oai -l nl
bash arthub-index.sh -e http://datahub.vlaamsekunstcollectie.be/oai -l en
```

This will export all Datahub records and create an Apache Solr document for each record. You can then browse to the Arthub in your browser to look them up.
