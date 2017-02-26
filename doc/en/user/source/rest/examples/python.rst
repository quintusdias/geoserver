.. _rest_examples_python:

Python
======

The examples in this section use `Python <https://www.python.org/>`_.
Python's standard library includes the `urllib <https://docs.python.org/3/library/urllib.request.html>`_ modules for making HTTP requests, but these examples will instead employ the third-party `Requests <http://docs.python-requests.org/>`_ module, which is often installed in addition to your Python distribution.
In addition, the standard library module `io <https://docs.python.org/3/library/io.html>`_, `os <https://docs.python.org/3/library/os.html>`_, and the third party library `lxml <http://lxml.de>`_ are used to process responses.
These examples do not use `gsconfig.py <https://github.com/dwins/gsconfig.py/wiki>`_. 

.. todo::

   The following extra sections could be added for completeness:

   * Deleting a workspace/store/featuretype/style/layergroup
   * Renaming a workspace/store/featuretype/style/layergroup

Setup
----------------------

The following code should be run before the examples.

.. code-block:: console

   # Standard library imports
   import io
   import os

   # Third party library imports
   import requests
   from lxml import etree

   s = requests.Session()
   s.auth = ('admin', 'geoserver')

Adding a new workspace
----------------------

The following creates a new workspace named "acme" with a POST request:

.. code-block:: console

   url = 'http://localhost:8080/geoserver/rest/workspaces.xml'
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

.. code-block:: console

   url = 'http://localhost:8080/geoserver/rest/workspaces/acme.xml'
   r = s.get(url)
   doc = etree.parse(io.BytesIO(r.content))
   etree.dump(doc.getroot())

The response should look like this:

.. code-block:: xml

   <workspace>
     <name>acme</name>
     <dataStores>
       <atom:link xmlns:atom="http://www.w3.org/2005/Atom" rel="alternate" href="http://localhost:8080/geoserver/rest/workspaces/acme/datastores.xml" type="application/xml"/>
     </dataStores>
     <coverageStores>
       <atom:link xmlns:atom="http://www.w3.org/2005/Atom" rel="alternate" href="http://localhost:8080/geoserver/rest/workspaces/acme/coveragestores.xml" type="application/xml"/>
     </coverageStores>
     <wmsStores>
       <atom:link xmlns:atom="http://www.w3.org/2005/Atom" rel="alternate" href="http://localhost:8080/geoserver/rest/workspaces/acme/wmsstores.xml" type="application/xml"/>
     </wmsStores>
   </workspace>

This shows that the workspace can contain "``dataStores``" (for :ref:`vector data <data_vector>`), "``coverageStores``" (for :ref:`raster data <data_raster>`), and "``wmsStores``" (for :ref:`cascaded WMS servers <data_external_wms>`).

Uploading a shapefile
---------------------

In this example a new store will be created by uploading a shapefile.

The following request uploads a zipped shapefile named ``roads.zip`` and creates a new store named ``roads``.

.. code-block:: console

   url = 'http://localhost:8080/geoserver/rest/workspaces/acme/datastores/roads/file.shp'
   headers = {'Content-Type': 'application/zip'}
   with open('roads.zip', 'rb') as f:
       data = f.read()
   r = s.put(url, headers=headers, data=data)
   print(r)

If executed correctly, the output should contain the following::

   <Response [201]>

The store information can be retrieved as XML with a GET request:

.. code-block:: console

   url = 'http://localhost:8080/geoserver/rest/workspaces/acme/datastores/roads.xml'
   r = s.get(url)
   doc = etree.parse(io.BytesIO(r.content))
   etree.dump(doc.getroot())

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

By default when a shapefile is uploaded, a feature type is automatically created. The feature type information can be retrieved as XML with a GET request:

.. code-block:: console

   url = 'http://localhost:8080geoserver/rest/workspaces/acme/datastores/roadsfeaturetypes/roads.xml'
   r = s.get(url)                                                                  
   doc = etree.parse(io.BytesIO(r.content))                                        
   etree.dump(doc.getroot())                                                       

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

Consider a directory ``/data/rivers`` that contains the shapefile ``rivers.shp``. The following adds a new store for the shapefile:

.. code-block:: console

   url = 'http://localhost:8080/geoserver/rest/workspaces/acme/datastores/rivers/external.shp'
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

.. code-block:: console

   url = 'http://localhost:8080/geoserver/rest/workspaces/acme/datastores/rivers.html'
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

.. code-block:: console

   url = 'http://localhost:8080/geoserver/rest/workspaces/acme/datastores/shapefiles/external.shp?configure=all'
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

.. code-block:: console

   url = 'http://localhost:8080/geoserver/rest/workspaces/acme/datastores/shapefiles.html'
   r = s.get(url)
   doc = etree.HTML(r.content)
   etree.dump(doc)

Adding a GeoTIFF Raster
-----------------------

This example shows how to load and create a store that contains a GeoTIFF.
Consider a GeoTIFF on the server ``/data/rasters/Baltic.tif``.  
First create a coveragestore for it.

.. code-block:: console

url = 'http://localhost:8080/geoserver/rest/workspaces/acme/coveragestores'
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

.. code-block:: console

   url = 'http://localhost:8080/geoserver/rest/workspaces/acme/coveragestores/Baltic/external.geotiff'
   headers = {'Content-Type': 'text/plain'}
   data = "file:///data/rasters/Baltic_sea.tif"
   r = s.put(url, headers=headers, data=data)
   print(r)

If executed correctly, the response should contain the following::
 
   <Response [201]>

The raster will be added to the existing store and published as a layer.

The coveragestore information can be retrieved as XML with a GET request:

.. code-block:: console

   url = 'http://localhost:8080/geoserver/rest/workspaces/acme/coveragestores/Baltic.xml'
   r = s.get(url)
   doc = etree.parse(io.BytesIO(r.content))
   etree.dump(doc.getroot())

Deleting a workspace
--------------------

This example shows how to delete a workspace and all its contents.
The "acme" store that has been populated throught these examples will
be deleted.

.. code-block:: console

   url = 'http://localhost:8080/geoserver/rest/workspaces/acme.xml'
   params = {'recurse': True}
   r = s.delete(url, params=params)
   print(r)

If executed correctly, the response should contain the following::
 
   <Response [200]>

