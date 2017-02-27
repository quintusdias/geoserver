.. _rest_examples_python:

Python
======

The examples in this section use `Python <https://www.python.org/>`_.
Python's standard library includes the `urllib <https://docs.python.org/3/library/urllib.request.html>`_ modules for making HTTP requests, but these examples will instead employ the third-party `Requests <http://docs.python-requests.org/>`_ module, which is often installed in addition to your Python distribution.
In addition, the third party library `lxml <http://lxml.de>`_ is used to process responses.
These examples do not use `gsconfig.py <https://github.com/dwins/gsconfig.py/wiki>`_. 

.. todo::

   The following extra sections could be added for completeness:

   * Deleting a store/featuretype/style/layergroup
   * Renaming a workspace/store/featuretype/style/layergroup
   * Configuring an available coverage
   * Uploading an app-schema mapping file
   * Uploading multiple app-schema mapping files
   * Creating a layer style
   * Changing a layer style
   * Creating a layer style (SLD package)
   * Adding a PostGIS table
   * Creating a layer group
   * Filtering over resource name
   * Filtering over resource properties
   * Uploading and modifying a image mosaic
   * Master Password Change

Setup
----------------------

The following code should be run before the examples.

.. code-block:: python

   # Standard library imports
   import os

   # Third party library imports
   import requests
   from lxml import etree

   s = requests.Session()
   s.auth = ('admin', 'geoserver')

Adding a new workspace
----------------------

The following creates a new workspace named ``acme`` with a POST request:

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'
          '/workspaces.xml')
   headers = {'Content-Type': 'text/xml'}
   data = "<workspace><name>acme</name></workspace>"
   r = s.post(url, headers=headers, data=data)
   print(r)
   print(r.headers)

If executed correctly, the output should contain the following::

   <Response [201]>
   ...
   'Location': 'http://localhost:8080/geoserver/rest/workspaces.xml/acme'

Note the ``Location`` response header, which specifies the location (URI) of the newly created workspace.

The workspace information can be retrieved as XML with a GET request:

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'
          '/workspaces/acme.xml')
   r = s.get(url)
   doc = etree.fromstring(r.content)
   etree.dump(doc)

The response should look like this:

.. code-block:: xml

   <workspace>
     <name>acme</name>
     <dataStores>
       <atom:link
          xmlns:atom="http://www.w3.org/2005/Atom"
          rel="alternate"
          href="http://localhost:8080/geoserver/rest/workspaces/acme/datastores.xml"
          type="application/xml"/>
     </dataStores>
     <coverageStores>
       <atom:link
          xmlns:atom="http://www.w3.org/2005/Atom"
          rel="alternate"
          href="http://localhost:8080/geoserver/rest/workspaces/acme/coveragestores.xml"
          type="application/xml"/>
     </coverageStores>
     <wmsStores>
       <atom:link
          xmlns:atom="http://www.w3.org/2005/Atom"
          rel="alternate"
          href="http://localhost:8080/geoserver/rest/workspaces/acme/wmsstores.xml"
          type="application/xml"/>
     </wmsStores>
   </workspace>

This shows that the workspace can contain "``dataStores``" (for :ref:`vector data <data_vector>`), "``coverageStores``" (for :ref:`raster data <data_raster>`), and "``wmsStores``" (for :ref:`cascaded WMS servers <data_external_wms>`).

Uploading a shapefile
---------------------

In this example a new store will be created by uploading a shapefile.

The following request uploads a zipped shapefile named ``roads.zip``
and creates a new store named ``roads``.

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'
          '/workspaces/acme/datastores/roads/file.shp')
   headers = {'Content-Type': 'application/zip'}
   with open('roads.zip', 'rb') as f:
       data = f.read()
   r = s.put(url, headers=headers, data=data)
   print(r)

If executed correctly, the output should contain the following::

   <Response [201]>

The store information can be retrieved as XML with a GET request:

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'
          '/workspaces/acme/datastores/roads.xml')
   r = s.get(url)
   doc = etree.fromstring(r.content)
   etree.dump(doc)

The response should look like this:

.. code-block:: xml

   <dataStore>
     <name>roads</name>
     <type>Shapefile</type>
     <enabled>true</enabled>
     <workspace>
       <name>acme</name>
       <atom:link xmlns:atom="http://www.w3.org/2005/Atom" rel="alternate" href="http://localhost:8080/geoserver/rest/workspaces/acme.xml" type="application/xml"/>
     </workspace>
     <connectionParameters>
       <entry key="namespace">http://acme</entry>
       <entry key="url">file:/var/lib/tomcat/webapps/geoserver/data/data/acme/roads/</entry>
     </connectionParameters>
     <__default>false</__default>
     <featureTypes>
       <atom:link xmlns:atom="http://www.w3.org/2005/Atom" rel="alternate" href="http://localhost:8080/geoserver/rest/workspaces/acme/datastores/roads/featuretypes.xml" type="application/xml"/>
     </featureTypes>
   </dataStore>

By default when a shapefile is uploaded, a feature type is automatically
created. The feature type information can be retrieved as XML with
a GET request:

.. code-block:: python

   url = ('http://localhost:8080geoserver/rest'
          '/workspaces/acme/datastores/roads'
          '/featuretypes/roads.xml')
   r = s.get(url)                                                                  
   doc = etree.fromstring(r.content)                                        
   etree.dump(doc)                                                       

If executed correctly, the response will be:

.. code-block:: xml

   <featureType>
     <name>roads</name>
     <nativeName>roads</nativeName>
     <namespace>
       <name>acme</name>
       <atom:link xmlns:atom="http://www.w3.org/2005/Atom" rel="alternate" href="http://localhost:8080/geoserver/rest/namespaces/acme.xml" type="application/xml"/>
     </namespace>
     ...
   </featureType>
   

Adding an existing shapefile
----------------------------

In the previous example a shapefile was uploaded directly to GeoServer
by sending a zip file in the body of a PUT request. This example shows
how to publish a shapefile that already exists on the server.

Consider a directory ``/data/rivers`` that contains the shapefile
``rivers.shp``. The following adds a new store for the shapefile:

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'
          '/workspaces/acme'
          '/datastores/rivers/external.shp')
   headers = {'Content-Type': 'text/plain'}
   data = "file:///data/rivers/rivers.shp"
   r = s.put(url, headers=headers, data=data)
   print(r)

The ``external.shp`` part of the request URI indicates that the file is coming from outside the catalog.

If executed correctly, the response should contain the following::
 
   <Response [201]>

The shapefile will be added to the existing store and published as a layer.

To verify the contents of the store, execute a GET request.  Since the
XML response only provides details about the store itself without showing
its contents, execute a GET request for HTML:

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'
          '/workspaces/acme/datastores/rivers.html')
   r = s.get(url)
   doc = etree.HTML(r.content)
   etree.dump(doc)

Adding a directory of existing shapefiles
-----------------------------------------

This example shows how to load and create a store that contains a number
of shapefiles, all with a single operation. This example is very similar
to the example above of adding a single shapefile.

Consider a directory on the server ``/data/shapefiles`` that contains
multiple shapefiles. The following adds a new store for the directory.

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'
          '/workspaces/acme'
          '/datastores/shapefiles/external.shp?configure=all')
   headers = {'Content-Type': 'text/plain'}
   data = "file:///data/shapefiles/"
   r = s.put(url, headers=headers, data=data)
   print(r)

Note the ``configure=all`` query string parameter, which sets each
shapefile in the directory to be loaded and published.

If executed correctly, the response should contain the following::
 
   <Response [201]>

To verify the contents of the store, execute a GET request.  Since the
XML response only provides details about the store itself without showing
its contents, execute a GET request for HTML:

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'
          '/workspaces/acme/datastores/shapefiles.html')
   r = s.get(url)
   doc = etree.HTML(r.content)
   etree.dump(doc)

Adding a GeoTIFF Raster
-----------------------

This example shows how to load and create a store that contains a GeoTIFF.
Consider a GeoTIFF on the server ``/data/rasters/Baltic.tif``.  
First create a coveragestore for it:

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'
          '/workspaces/acme/coveragestores')
   data = """<coverageStore>
               <name>Baltic</name>
               <workspace>acme</workspace>
               <enabled>true</enabled>
             </coverageStore>"""
   headers = {'Content-Type': 'text/xml'}
   r = s.post(url, headers=headers, data=data)
   print(r)

If executed correctly, the response should contain the following::
 
   <Response [201]>

Now load the GeoTIFF itself.

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'
          '/workspaces/acme'
          '/coveragestores/Baltic/external.geotiff')
   headers = {'Content-Type': 'text/plain'}
   data = "file:///data/rasters/Baltic_sea.tif"
   r = s.put(url, headers=headers, data=data)
   print(r)

If executed correctly, the response should contain the following::
 
   <Response [201]>

The raster will be added to the existing store and published as a layer.

The coveragestore information can be retrieved as XML with a GET request:

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'
          '/workspaces/acme/coveragestores/Baltic.xml')
   r = s.get(url)
   doc = etree.fromstring(r.content)
   etree.dump(doc)

Adding a PostGIS database
-------------------------

In this example a PostGIS database named ``nyc`` will be added as
a new store. This section assumes that a PostGIS database named
``nyc`` is present on the local system and is accessible by the
user ``bob``.

.. code-block:: python

   data = """<dataStore>                                                              
     <name>nyc</name>                                                                 
     <connectionParameters>                                                           
       <host>localhost</host>                                                         
       <port>5432</port>                                                              
       <database>nyc</database>                                                       
       <user>bob</user>                                                               
       <passwd>postgres</passwd>                                                      
       <dbtype>postgis</dbtype>                                                       
     </connectionParameters>                                                          
   </dataStore>"""                                                                    
   url = ('http://localhost:8080/geoserver/rest'
          '/workspaces/acme/datastores')
   headers = {'Content-Type': 'text/xml'}
   r = s.post(url, headers=headers, data=data)
   print(r)

If executed correctly, the response should contain the following::
 
   <Response [201]>

The store information can be retrieved as XML with a GET request:

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'
          '/workspaces/acme/datastores/nyc.xml')
   r = s.get(url)                                                                     
   doc = etree.fromstring(r.content)                                           
   etree.dump(doc)  

The store information can be retrieved as XML with a GET request:

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'
          '/workspaces/acme/datastores/nyc.xml')
   r = s.get(url)
   doc = etree.fromstring(r.content)
   etree.dump(doc)

The response should look like the following:

.. code-block:: xml

   <dataStore>
     <name>nyc</name>
     <type>PostGIS</type>
     <enabled>true</enabled>
     <workspace>
       <name>acme</name>
       <atom:link xmlns:atom="http://www.w3.org/2005/Atom" rel="alternate" href="http://localhost:8080/geoserver/rest/workspaces/acme.xml" type="application/xml"/>
     </workspace>
     <connectionParameters>
       <entry key="database">nyc</entry>
       <entry key="port">5432</entry>
       <entry key="passwd">crypt1:iN+oI8QeT+R8tpecSoRLLGX+igST5oiy</entry>
       <entry key="host">localhost</entry>
       <entry key="dbtype">postgis</entry>
       <entry key="namespace">http://acme</entry>
       <entry key="user">bob</entry>
     </connectionParameters>
     <__default>false</__default>
     <featureTypes>
       <atom:link xmlns:atom="http://www.w3.org/2005/Atom" rel="alternate" href="http://localhost:8080/geoserver/rest/workspaces/acme/datastores/nyc/featuretypes.xml" type="application/xml"/>
     </featureTypes>
   </dataStore>

Creating a PostGIS table
------------------------

This example will not only create a new feature type in GeoServer,
but will also create the PostGIS table itself.

This request will perform the feature type creation and add the new table:

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'
          '/workspaces/acme/datastores/nyc/featuretypes')           
   headers = {'Content-Type': 'text/xml'}                                          

   data = """<featureType>                                                         
     <name>annotations</name>                                                      
     <nativeName>annotations</nativeName>                                          
     <title>Annotations</title>                                                    
     <srs>EPSG:4326</srs>                                                          
     <attributes>                                                                  
       <attribute>                                                                 
         <name>the_geom</name>                                                     
         <binding>com.vividsolutions.jts.geom.Point</binding>                      
       </attribute>                                                                
       <attribute>                                                                 
         <name>description</name>                                                  
         <binding>java.lang.String</binding>                                       
       </attribute>                                                                
       <attribute>                                                                 
         <name>timestamp</name>                                                    
         <binding>java.util.Date</binding>                                         
       </attribute>                                                                
     </attributes>                                                                 
   </featureType>"""                                                               

   r = s.post(url, data=data, headers=headers)                                     
   print(r)  
    
The result is a new, empty table named "annotations" in the "nyc"
database, fully configured as a feature type.

The featuretype information can be retrieved as XML with a GET request:

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'                                   
          '/workspaces/acme/datastores/nyc/featuretypes/annotations.xml')          
   r = s.get(url)                                                                  
   print(r)                                                                        
   doc = etree.fromstring(r.content)
   etree.dump(doc) 

Retrieving component versions
-----------------------------

This example shows how to retrieve the versions of the main components:
GeoServer, GeoTools, and GeoWebCache:

.. code-block:: python

   url = 'http://localhost:8080/geoserver/rest/about/version.xml'
   r = s.get(url)                                                                  
   doc = etree.fromstring(r.content)
   etree.dump(doc) 

The response will look something like this:

.. code-block:: xml

   <about>
     <resource name="GeoServer">
       <Build-Timestamp>20-Dec-2016 17:31</Build-Timestamp>
       <Version>2.10.1</Version>
       <Git-Revision>46d8beb44231642944599962b58ee0cccd03fcbb</Git-Revision>
     </resource>
     <resource name="GeoTools">
       <Build-Timestamp>19-Dec-2016 22:01</Build-Timestamp>
       <Version>16.1</Version>
       <Git-Revision>c4fcd240049fa0506bb17c9e2281fc963bc9b51a</Git-Revision>
     </resource>
     <resource name="GeoWebCache">
       <Version>1.10.1</Version>
       <Git-Revision>1.10.x/0355b0eb5a5f2a95f387ce5c30cdf2548ffb1744</Git-Revision>
     </resource>
   </about>

Retrieving manifests
--------------------

This collection of examples shows how to retrieve the full manifest
and subsets of the manifest as known to the ClassLoader.


.. code-block:: python

   url = 'http://localhost:8080/geoserver/rest/about/manifest.xml'
   r = s.get(url)                                                                  
   doc = etree.fromstring(r.content)
   etree.dump(doc) 

The result will be a very long list of manifest information. While
this can be useful, it is often desirable to filter this list.

Creating an empty mosaic and harvest granules
---------------------------------------------

The next command uploads an :download:`empty.zip` file. 
This archive contains the definition of an empty mosaic (no granules in this case) through the following files::

      datastore.properties (the postgis datastore connection params)
      indexer.xml (The mosaic Indexer, note the CanBeEmpty=true parameter)
      polyphemus-test.xml (The auxiliary file used by the NetCDF reader to parse schemas and tables)

.. note:: **Make sure to update the datastore.properties file** with your connection params and refresh the zip when done, before uploading it. 
.. note:: The code blocks below contain a single command that is extended over multiple lines.
.. note:: The configure=none parameter allows for future configuration after harvesting

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'                                   
          '/workspaces/topp'
          '/coveragestores/empty/file.imagemosaic?configure=none') 
   headers = { 'Content-Type': 'application/zip', }                                
   with open('empty.zip', 'rb') as f:                                         
       data = f.read()                                                             
   r = s.put(url, headers=headers, data=data)                                      
   r.raise_for_status()                                                            
   print(r)  

The following instead instructs the mosaic to harvest a single :download:`polyphemus_20120401.nc` file into the mosaic, collecting its properties and updating the mosaic index:

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'                                   
          '/workspaces/topp'
          '/coveragestores/empty/external.imagemosaic') 
   headers = { 'Content-Type': 'text/plain', }                                
   data = "file:///path/to/polyphemus_20120401.nc"
   r = s.post(url, headers=headers, data=data)                                      
   print(r) 

Once done you can get the list of coverages/granules available on that store.

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'                                   
          '/workspaces/topp/coveragestores/empty/coverages.xml')
   params = {'list': 'all'}
   r = s.get(url, params=params)
   doc = etree.fromstring(r.content)
   etree.dump(doc)

which will result in the following:

.. code-block:: xml

      <list>
        <coverageName>NO2</coverageName>
        <coverageName>O3</coverageName>
      </list>


Deleting a workspace
--------------------

This example shows how to delete a workspace and all its contents.
The "acme" store that has been populated throught these examples will
be deleted.

.. code-block:: python

   url = ('http://localhost:8080/geoserver/rest'
          '/workspaces/acme.xml')
   params = {'recurse': True}
   r = s.delete(url, params=params)
   print(r)

If executed correctly, the response should contain the following::
 
   <Response [200]>

