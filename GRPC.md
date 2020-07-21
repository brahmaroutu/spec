```protobuf
 syntax = "proto3";

 package cosi.v1;

 import "google/protobuf/descriptor.proto";

 option go_package = "github.com/container-object-store-interface/go-cosi";

 extend google.protobuf.MessageOptions {

     // cosi_secret should be used to designate messages containing sensitive data
     //             to provide protection against that data being logged or otherwise leaked.
     bool cosi_secret = 1000;
 }

 message DriverInfoRequest {
     // INTENTIONALLY BLANK
 }

 // DataProtocol defines a set of constants used in Create and Grant requests.
 enum DataProtocol {
     PROTOCOL_UNSPECIFIED = 0;
     AZURE_BLOB = 1;
     GCS = 2;
     S3 = 3;
 }

 // AccessMode defines a common set of permissions among object stores
 enum AccessMode {
     MODE_UNSPECIFIED = 0;
     RO = 1;
     WO = 2;
     RW = 3;
 }

 // S3SignatureVersion defines the 2 supported versions of S3's authentication
 enum S3SignatureVersion {
     VERSION_UNSPECIFIED = 0;
     V2 = 1;
     V4 = 2;
 }

 message DriverInfoResponse {
     // DriverName
     string DriverName = 1;

     // SupportedProtocols
     repeated DataProtocol SupportedProtocols = 2;

     // NextId = 3;
 }
 
 message s3_context {
     // returns the location where bucket will be created
     string location 
 }

message gcs_context {
   // returns the location where bucket is created
   string location
   // returns the project the bucket belongs to
   string project
}

message azure_context {
}

message generic_context {
     // generic output content
     map<string, string> bucket_data
}

message Bucket {
  // Name is the name of the bucket
  string name = 1
  // provisioner used to create and other bucket operations
  // ProjectName this bucket created under
  ProviderContext provider_context = 2 - { azure_context, gcs_context, s3_context, generic_context}
  string provisioner = 3
  // access mode
  AccessMode access_mode = 4;
    
}


 message CreateBucketRequest {
     // bucket_name, This field is REQUIRED.
     // Maintain Idempotency. 
     //    In the case of error, the CO MUST handle the gRPC error codes
     //    per the recovery behavior defined in the "CreateBucket Errors"
     //    section below.
     // BucketRequest:name 
     string bucket_name = 1;

     // RequestProtocol, one of the predefined values
     // Driver must check the protocol used to match 
     // BucketClass:supportedProtocols - {"azureblob", "gcs", "s3", ... } [3]
     // BucketRequest:protocol - use this as request protocol but check 
     // if the protocol is in the BucketClass' suppportedProtocols
     DataProtocol request_protocol = 2;

     // DriverParameters, these are parameters that are extracted from 
     // BuckerRequest and BucketClass so that the call has context.
     // For example GCS require projectName for CreateBucket to succeed.
     // BucketClass:provisioner - identify the  
     // projectID if GCS
     map<string, string> driver_parameters = 3;  //should be similar to ProviderContext

     // AccessMode is requested as RO, RW, WO and depends on driver.
     // If driver supports access mode if not ignores it
     // BucketClass:accessMode - {"ro", "wo", "rw"} [4]
     AccessMode access_mode = 4;

     // Information required to make createBucket call. This field is REQUIRED
     // A series of tokens, user name, etc based on protocol choice
     // BucketRequest:secretName will provide necessary security token to
     // connect to the provider API.  
     // Azure:
     //    message AuthenticationData {
     //        option (cosi_secret) = true;
     //        string StorageAccountName = 1;
     //        string AccountKey = 2;
     //        string SasToken = 3;
     //    }
     // GCS:
     //    message AuthenticationData {
     //        option (cosi_secret) = true;
     //        string StorageAccountName = 1;
     //        string PrivateKeyName = 2;
     //        string PrivateKey = 3;
     //    }
     // S3:
     //    message AuthenticationData {
     //        option (cosi_secret) = true;
     //        string AccessKeyId = 1;
     //        string SecretKey = 2;
     //        string StsToken = 3;
     //        string UserName = 4;
     //    }
     map<string, string> secrets = 5;
 }


CREATE_INVALID_ARGUMENT    : validation of the input argument fails 
CREATE_INVALID_PROTOCOL    : driver does not support the protocol
CREATE_ALREADY_EXISTS      : resource already exists 
CREATE_INVALID_CREDENTIALS : resource creation failed due to invalid credentials 
CREATE_INTERNAL_ERROR      : Failed to execute the requested call

 message CreateBucketResponse {
     // Bucket returned
     Bucket bucket
 }
 
 message DeleteBucketRequest {
     // The name of the bucket to be deleted.
     // This field is REQUIRED.
     string bucket_name = 1;
    
     // RequestProtocol, one of the predefined values
     // Driver must check the protocol used to match 
     // BucketClass:supportedProtocols - {"azureblob", "gcs", "s3", ... } [3]
     // BucketRequest:protocol - use this as request protocol but check 
     // if the protocol is in the BucketClass' suppportedProtocols
     DataProtocol request_protocol = 2;

     // DriverParameters, these are parameters that are extracted from 
     // BuckerRequest and BucketClass so that the call has context.
     // For example GCS require projectName for CreateBucket to succeed.
     // BucketClass:provisioner - identify the  
     // projectID if GCS
     map<string, string> driver_parameters = 3; //provider_context
  
     // Secrets required by driver to complete bucket deletion request.
     // This field is OPTIONAL. Refer to the `Secrets Requirements`
     // section on how to use this field.
     // Azure:
     //    message AuthenticationData {
     //        option (cosi_secret) = true;
     //        string StorageAccountName = 1;
     //        string AccountKey = 2;
     //        string SasToken = 3;
     //    }
     // GCS:
     //    message AuthenticationData {
     //        option (cosi_secret) = true;
     //        string StorageAccountName = 1;
     //        string PrivateKeyName = 2;
     //        string PrivateKey = 3;
     //    }
     // S3:
     //    message AuthenticationData {
     //        option (cosi_secret) = true;
     //        string AccessKeyId = 1;
     //        string SecretKey = 2;
     //        string StsToken = 3;
     //        string UserName = 4;
     //    }
     map<string, string> secrets = 2 
}



 message DeleteBucketResponse {
     // INTENTIONALLY BLANK
 }

DELETE_BUCKET_DOESNOT_EXIST : Bucket specified does not exist 
DELETE_DELETE_INPROGRESS    : Delete bucket is in progress
DELETE_INVALID_CREDENTIALS  : resource deletion failed due to invalid credentials
DELETE_INTERNAL_ERROR       : Failed to execute the requested call
DELETE_INVALID_ARGUMENT     : validation of the input argument fails 



service CosiController {
  rpc CreateBucket (CreateBucketRequest)
    returns (CreateBucketResponse) {}

  rpc DeleteBucket (DeleteBucketRequest)
    returns (DeleteBucketResponse) {}
}



 message GrantBucketAccessRequest {
     // The name of the bucket to be granted access.
     // This field is REQUIRED.
     string bucket_name = 1;
    
     // RequestProtocol, one of the predefined values
     // Driver must check the protocol used to match 
     // BucketClass:supportedProtocols - {"azureblob", "gcs", "s3", ... } [3]
     // BucketRequest:protocol - use this as request protocol but check 
     // if the protocol is in the BucketClass' suppportedProtocols
     DataProtocol request_protocol = 2;

     // principals that are granted permission
     repeated string principalID

     // permission granted
     string permission
 }

 message GrantBucketAccessResponse {
     // No data returned by this call other than error or success code
 }
 
 
GRANT_BUCKET_DOESNOT_EXIST : Bucket specified does not exist 
GRANT_INVALID_CREDENTIALS  : resource deletion failed due to invalid credentials
GRANT_INTERNAL_ERROR       : Failed to execute the requested call
GRANT_INVALID_ARGUMENT     : validation of the input argument fails
GRANT_INVALID_PRINCIPAL    : Failed to grant, principal provided is invalid 
 
 

 message RevokeBucketAccessRequest {
     // The name of the bucket to be granted access.
     // This field is REQUIRED.
     string bucket_name = 1;
    
     // RequestProtocol, one of the predefined values
     // Driver must check the protocol used to match 
     // BucketClass:supportedProtocols - {"azureblob", "gcs", "s3", ... } [3]
     // BucketRequest:protocol - use this as request protocol but check 
     // if the protocol is in the BucketClass' suppportedProtocols
     DataProtocol request_protocol = 2;

     // principals that are being revoke permission
     repeated string principalID

     // permission revoked
     string permission

 }

 message RevokeBucketAccessResponse {
     // may be return list of acl. For now left blank
 }

REVOKE_BUCKET_DOESNOT_EXIST : Bucket specified does not exist 
REVOKE_INVALID_CREDENTIALS  : resource deletion failed due to invalid credentials
REVOKE_INTERNAL_ERROR       : Failed to execute the requested call
REVOKE_INVALID_ARGUMENT     : validation of the input argument fails
REVOKE_INVALID_PRINCIPAL    : Failed to grant, principal provided is invalid 



 service CosiController {
     rpc GrantBucketAccess (GrantBucketAccessRequest) returns (GrantBucketAccessResponse);
     rpc RevokeBucketAccess (RevokeBucketAccessRequest) returns (RevokeBucketAccessResponse);
}

