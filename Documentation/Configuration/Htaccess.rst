htaccess file
^^^^^^^^^^^^^

This is the base .htaccess configuration. Please take a look for the default variables (SFC_ROOT, SFC_GZIP) and read the comments carefully.

.. code-block:: bash

   ### Begin: StaticFileCache (preparation) ####

   # Document root configuration
   RewriteRule .* - [E=SFC_ROOT:%{DOCUMENT_ROOT}]
   # RewriteRule .* - [E=SFC_ROOT:%{DOCUMENT_ROOT}/t3site] # Example if your installation is installed in a directory

   # Cleanup URI
   RewriteCond %{REQUEST_URI} ^.*$
   RewriteRule .* - [E=SFC_URI:/%{REQUEST_URI}]
   RewriteCond %{REQUEST_URI} ^/.*$
   RewriteRule .* - [E=SFC_URI:%{REQUEST_URI}]
   RewriteCond %{REQUEST_URI} ^/?$
   RewriteRule .* - [E=SFC_URI:/]

   # Cleanup HOST
   RewriteCond %{HTTP_HOST} ^([^:]+)(:[0-9]+)?$
   RewriteRule .* - [E=SFC_HOST:%1]

   # Get scheme
   RewriteRule .* - [E=SFC_PROTOCOL:http]
   RewriteCond %{SERVER_PORT} ^443$ [OR]
   RewriteCond %{HTTP:X-Forwarded-Proto} https
   RewriteRule .* - [E=SFC_PROTOCOL:https]

   # Get port
   RewriteRule .* - [E=SFC_PORT:80]
   RewriteCond %{ENV:SFC_PROTOCOL} ^https$ [NC]
   RewriteRule .* - [E=SFC_PORT:443]
   RewriteCond %{SERVER_PORT} ^[0-9]*$
   RewriteRule .* - [E=SFC_PORT:%{SERVER_PORT}]

    # Full path for redirect
   RewriteRule .* - [E=SFC_FULLPATH:typo3temp/tx_staticfilecache/%{ENV:SFC_PROTOCOL}/%{ENV:SFC_HOST}/%{ENV:SFC_PORT}%{ENV:SFC_URI}]

   # Check if the requested file exists in the cache, otherwise default to index.html that
   # set in an environment variable that is used later on
   # Note: With old RealURL structure perhaps you have to remove the "/" https://github.com/lochmueller/staticfilecache/pull/90
   # Note: We cannot check realurl "appendMissingSlash" or other BE related settings here - in front of the delivery.
   # Perhaps you have to check the "SFC_FILE" value and set it to your related configution e.g. "index.html" (without leading slash).
   # More information at: https://github.com/lochmueller/staticfilecache/pull/28
   RewriteCond %{ENV:SFC_ROOT}/%{ENV:SFC_FULLPATH} !-f
   RewriteRule .* - [E=SFC_FULLPATH:%{ENV:SFC_FULLPATH}/index.html]

   # Extension (Order: br, gzip, default)
   RewriteRule .* - [E=SFC_EXT:]
   RewriteCond %{HTTP:Accept-Encoding} br [NC]
   RewriteRule .* - [E=SFC_EXT:.br]
   RewriteCond %{ENV:SFC_ROOT}/%{ENV:SFC_FULLPATH}%{ENV:SFC_EXT} !-f
   RewriteRule .* - [E=SFC_EXT:]
   RewriteCond %{ENV:SFC_EXT} ^$
   RewriteCond %{HTTP:Accept-Encoding} gzip [NC]
   RewriteRule .* - [E=SFC_EXT:.gz]
   RewriteCond %{ENV:SFC_EXT} ^\.gz$
   RewriteCond %{ENV:SFC_ROOT}/%{ENV:SFC_FULLPATH}%{ENV:SFC_EXT} !-f
   RewriteRule .* - [E=SFC_EXT:]

   # Write Extension to SFC_FULLPATH
   RewriteRule .* - [E=SFC_FULLPATH:%{ENV:SFC_FULLPATH}%{ENV:SFC_EXT}]

   ### Begin: StaticFileCache (main) ####

   # We only redirect URI's without query strings
   RewriteCond %{QUERY_STRING} ^$

   # It only makes sense to do the other checks if a static file actually exists.
   RewriteCond %{ENV:SFC_ROOT}/%{ENV:SFC_FULLPATH} -f

   # NO frontend or backend user is logged in. Logged in users may see different
   # information than anonymous users. But the anonymous version is cached. So
   # don't show the anonymous version to logged in users.
   RewriteCond %{HTTP_COOKIE} !staticfilecache [NC]

   # We only redirect GET requests
   RewriteCond %{REQUEST_METHOD} GET

   # Rewrite the request to the static file.
   RewriteRule .* %{ENV:SFC_ROOT}/%{ENV:SFC_FULLPATH} [L]

   # Do not allow direct call the cache entries
   RewriteCond %{ENV:SFC_URI} ^/typo3temp/tx_staticfilecache/.*
   RewriteCond %{ENV:REDIRECT_STATUS} ^$
   RewriteRule .* - [F,L]

   ### Begin: StaticFileCache (options) ####

   # Set proper content type and encoding for gzipped html.
   <FilesMatch "\.gz">
      SetEnv no-gzip 1
      <IfModule mod_headers.c>
         Header set Content-Encoding gzip
      </IfModule>
   </FilesMatch>

   # if there are same problems with ForceType, please try the AddType alternative
   # Set proper content type gzipped html
   <FilesMatch "\.html\.gz">
      ForceType text/html
      # AddType "text/html" .gz
   </FilesMatch>
   <FilesMatch "\.xml\.gz">
      ForceType text/xml
      # AddType "text/xml" .gz
   </FilesMatch>
   <FilesMatch "\.rss\.gz">
      ForceType text/xml
      # AddType "text/xml" .gz
   </FilesMatch>

   ### End: StaticFileCache ###


If you use the oldschool .htaccess rewrite rules that come with the TYPO3 dummy, then the relevant StaticFileCache configuration should be inserted in the .htaccess file just before these lines:

.. code-block:: bash

   RewriteCond %{REQUEST_FILENAME} !-f
   RewriteCond %{REQUEST_FILENAME} !-d
   RewriteCond %{REQUEST_FILENAME} !-l
   RewriteRule .* index.php [L]

If the TYPO3 Installation isn´t in your root directory (say your site lives in http://some.domain.com/t3site/), then you have to add the '/t3site' part to the configuration snippet. It must be placed right after %{DOCUMENT_ROOT}. Here is the line of the ruleset to illustrate:

.. code-block:: bash

   RewriteRule .* - [E=SFC_ROOT:%{DOCUMENT_ROOT}/t3site]

You are of course free to make the rules as complex as you like.

There might be some files you never want to pull from cache even if they are indexed. For example you might have some custom realurl rules that make your RSS feed accessible as rss.xml. You can skip rewriting to static file with the following condition:

.. code-block:: bash

   RewriteCond %{REQUEST_FILENAME} !^.*\.xml$

Keep in mind: If you are using the gzip feature of StaticFileCache you have to take care, that the output is not encoded twice. If the result of the page are cryptic chars like "�‹��í[krÛH’þ-Eô�ª¹±-¹[ À—�É${dùÙkÙ�[îé..." remove the "text/html \" in the mod_deflate section of the default TYPO3 .htaccess rules.
