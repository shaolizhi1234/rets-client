rets-client
===========
A RETS (Real Estate Transaction Standard) client for Node.js.

Version 2.x of rets-client has a completely different interface from the 1.x version -- code written for 1.x will not
work with 2.x.  If you wish to continue to use the 1.x version, you can use the
[v1 branch](https://github.com/sbruno81/rets-client/tree/v1).

This interface uses promises, and future development plans include an optional stream-based interface
for better performance with large datasets and/or large objects.

This library is written primarily in CoffeeScript, but may be used just as easily in a Node app using Javascript or
CoffeeScript.  Promises in this module are provided by [Bluebird](https://github.com/petkaantonov/bluebird).

The original module was developed against a server running RETS v1.7.2, so there may be incompatibilities with other
versions.  However, we want this library to work against any RETS servers that are in current use, so issue tickets
describing problems or (even better) pull requests that fix interactions with servers running other versions of RETS
are welcomed.

[RETS Specifications](http://www.reso.org/specifications)

## Contributions
Issue tickets and pull requests are welcome.  Pull requests must be backward-compatible to be considered, and ideally
should match existing code style.

#### TODO
- create optional streaming interface 
- create unit tests -- specifically ones that run off example RETS data rather than requiring access to a real RETS server


## Example Usage

##### Client Configuration
```javascript
    //create rets-client
    var clientSettings = {
        loginUrl:retsLoginUrl,
        username:retsUser,
        password:retsPassword,
        version:'RETS/1.7.2',
        userAgent:'RETS node-client/1.0'
    };
...
```    
##### Client Configuration with UA Authorization
```javascript
    //create rets-client
    var clientSettings = {
        version:'RETS/1.7.2',
        userAgent:userAgent,
        userAgentPassword:userAgentPassword,
        sessionId:sessionId
    };
...
```

#### Example RETS Session
```javascript
  var rets = require('rets-client');
  var fs = require('fs');
  var photoId = '12345'; // <--- dummy example ID!
  var outputFields = function(obj, fields) {
    for (var i=0; i<fields.length; i++) {
      console.log(fields[i]+": "+obj[fields[i]]);
    }
    console.log("");
  };
  // establish connection to RETS server which auto-logs out when we're done
  rets.getAutoLogoutClient(clientSettings, function (client) {
    //get resources metadata
    return client.metadata.getResources()
      .then(function (data) {
        console.log("======================================");
        console.log("========  Resources Metadata  ========");
        console.log("======================================");
        outputFields(data, ['Version', 'Date']);
        for (var dataItem = 0; dataItem < data.results.length; dataItem++) {
          console.log("-------- Resource " + dataItem + " --------");
          outputFields(data.results[dataItem], ['ResourceID', 'StandardName', 'VisibleName', 'ObjectVersion']);
        }
      }).then(function () {
        //get class metadata
        return client.metadata.getClass("Property");
      }).then(function (data) {
        console.log("===========================================================");
        console.log("========  Class Metadata (from Property Resource)  ========");
        console.log("===========================================================");
        outputFields(data, ['Version', 'Date', 'Resource']);
        for (var classItem = 0; classItem < data.results.length; classItem++) {
          console.log("-------- Table " + classItem + " --------");
          outputFields(data.results[classItem], ['ClassName', 'StandardName', 'VisibleName', 'TableVersion']);
        }
      }).then(function () {
        //get field data for open houses
        return client.metadata.getTable("OpenHouse", "OPENHOUSE");
      }).then(function (data) {
        console.log("=============================================");
        console.log("========  OpenHouse Table Metadata  ========");
        console.log("=============================================");
        outputFields(data, ['Version', 'Date', 'Resource', 'Class']);
        for (var tableItem = 0; tableItem < data.results.length; tableItem++) {
          console.log("-------- Field " + tableItem + " --------");
          outputFields(data.results[tableItem], ['MetadataEntryID', 'SystemName', 'ShortName', 'LongName', 'DataType']);
        }
        return data.results
      }).then(function (fieldsData) {
        var plucked = [];
        for (var fieldItem = 0; fieldItem < fieldsData.length; fieldItem++) {
          plucked.push(fieldsData[fieldItem].SystemName);
        }
        return plucked;
      }).then(function (fields) {
        //perform a query using DQML2 -- pass resource, class, and query, and options
        return client.search.query("OpenHouse", "OPENHOUSE", "(OpenHouseType=PUBLIC),(ActiveYN=1)", {limit:100, offset:1})
        .then(function (searchData) {
          console.log("===========================================");
          console.log("========  OpenHouse Query Results  ========");
          console.log("===========================================");
          console.log("");
          //iterate through search results
          for (var dataItem = 0; dataItem < searchData.results.length; dataItem++) {
            console.log("-------- Result " + dataItem + " --------");
            outputFields(searchData.results[dataItem], fields);
          }
          if (searchData.maxRowsExceeded) {
            console.log("-------- More rows available!");
          }
        });
      }).then(function () {
        // get photos
        return client.objects.getPhotos("Property", "LargePhoto", photoId)
      }).then(function (photoList) {
        console.log("=================================");
        console.log("========  Photo Results  ========");
        console.log("=================================");
        for (var i = 0; i < photoList.length; i++) {
          console.log("Photo " + (i + 1) + " MIME type: " + photoList[i].mime);
          fs.writeFileSync(
            "/tmp/photo" + (i + 1) + "." + photoList[i].mime.match(/\w+\/(\w+)/i)[1],
            photoList[i].buffer
          );
        }
      });
  }).catch(function (error) {
    console.log("ERROR: issue encountered: "+(error.stack||error));
  });
```
