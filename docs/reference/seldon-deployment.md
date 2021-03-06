# Seldon Deployment

A SeldonDeployment is defined as a custom resource definition within Kubernetes.

 * [Design](#design)
 * [Proto Buffer Definiton](#definition)
 * [Examples](#examples)

## Design

![graph](./seldon-deployment-sketch.png)


## Proto Buffer Definition
The Seldon Deployment Custom Resource is defined using Proto Buffers. 

```proto
syntax = "proto2";
package seldon.protos;

import "k8s.io/apimachinery/pkg/apis/meta/v1/generated.proto";
import "v1.proto";

option java_package = "io.seldon.protos";
option java_outer_classname = "DeploymentProtos";

message SeldonDeployment {
  required string apiVersion = 1;
  required string kind = 2;
  optional k8s.io.apimachinery.pkg.apis.meta.v1.ObjectMeta metadata = 3;
  required DeploymentSpec spec = 4;
  optional DeploymentStatus status = 5;
}

/**
 * Status for seldon deployment
 */
message DeploymentStatus {
  optional string state = 1; // A short status value for the deployment.
  optional string description = 2; // A longer description describing the current state.
  repeated PredictorStatus predictorStatus = 3; // A list of individual statuses for each running predictor.
}

message PredictorStatus {
  required string name = 1; // The name of the predictor.
  optional string status = 2;  // A short status value.
  optional string description = 3; // A longer description of the current status.
  optional int32 replicas = 4; // The number of replicas requested.
  optional int32 replicasAvailable = 5; // The number of replicas available.
}


message DeploymentSpec {
  optional string name = 1; // A unique name within the namespace.
  repeated PredictorSpec predictors = 2; // A list of 1 or more predictors describing runtime machine learning deployment graphs.
  optional string oauth_key = 6; // The oauth key for external users to use this deployment via an API.
  optional string oauth_secret = 7; // The oauth secret for external users to use this deployment via an API.
  map<string,string> annotations = 8; // Arbitrary annotations.
}

message PredictorSpec {
  required string name = 1; // A unique name not used by any other predictor in the deployment.
  required PredictiveUnit graph = 2; // A graph describing how the predictive units are connected together.
  required k8s.io.api.core.v1.PodTemplateSpec componentSpec = 3; // A description of the set of containers used by the graph. One for each microservice defined in the graph.
  optional int32 replicas = 4; // The number of replicas of the predictor to create.
  map<string,string> annotations = 5; // Arbitrary annotations.
}


/**
 * Represents a unit in a runtime prediction graph that performs a piece of functionality within the prediction request/response calls.
 */
message PredictiveUnit {

  /**
   * The main type of the predictive unit. Routers decide where requests are sent, e.g. AB Tests and Multi-Armed Bandits. Combiners ensemble responses from their children. Models are leaft nodes in the predictive tree and provide request/reponse functionality encapsulating a machine learning model. Transformers alter the request features.
   */
  enum PredictiveUnitType {
    // Each one of these defines a default combination of Predictive Unit Methods
    UNKNOWN_TYPE = 0;
    ROUTER = 1; // Route + send feedback
    COMBINER = 2; // Aggregate
    MODEL = 3; // Transform input
    TRANSFORMER = 4; // Transform input (alias)
    OUTPUT_TRANSFORMER = 5; // Transform output
  }

  enum PredictiveUnitImplementation {
    // Each one of these are hardcoded in the engine, no microservice is used
    UNKNOWN_IMPLEMENTATION = 0; // No implementation (microservice used)
    SIMPLE_MODEL = 1; // An internal model stub for testing.
    SIMPLE_ROUTER = 2; // An internal router for testing.
    RANDOM_ABTEST = 3; // A A-B test that sends traffic 50% to each child randomly.
    AVERAGE_COMBINER = 4; // A default combiner that returns the average of its children responses.
  }

  enum PredictiveUnitMethod {
    TRANSFORM_INPUT = 0;
    TRANSFORM_OUTPUT = 1;
    ROUTE = 2;
    AGGREGATE = 3;
    SEND_FEEDBACK = 4;
  }

  required string name = 1; //must match container name of component if no implementation
  repeated PredictiveUnit children = 2; // The child predictive units.
  optional PredictiveUnitType type = 3;
  optional PredictiveUnitImplementation implementation = 4;
  repeated PredictiveUnitMethod methods = 5;
  optional Endpoint endpoint = 6; // The exposed endpoint for this unit.
  repeated Parameter parameters = 7; // Customer parameter to pass to the unit.
}

message Endpoint {

  enum EndpointType {
    REST = 0; // REST endpoints with JSON payloads
    GRPC = 1; // gRPC endpoints
  }

  optional string service_host = 1; // Hostname for endpoint.
  optional int32 service_port = 2; // The port to connect to the service.
  optional EndpointType type = 3; // The protocol handled by the endpoint.
}

message Parameter {

  enum ParameterType {
    INT = 0;
    FLOAT = 1;
    DOUBLE = 2;
    STRING = 3;
    BOOL = 4;
  }  

  required string name = 1;
  required string value = 2;
  required ParameterType type = 3;

}


```


## Examples

## Single Model

```json
{
    "apiVersion": "machinelearning.seldon.io/v1alpha1",
    "kind": "SeldonDeployment",
    "metadata": {
        "labels": {
            "app": "seldon"
        },
        "name": "seldon-deployment-example"
    },
    "spec": {
        "annotations": {
            "project_name": "FX Market Prediction",
            "deployment_version": "v1"
        },
        "name": "test-deployment",
        "oauth_key": "oauth-key",
        "oauth_secret": "oauth-secret",
        "predictors": [
            {
                "componentSpec": {
                    "spec": {
                        "containers": [
                            {
                                "image": "seldonio/mean_classifier:0.6",
                                "imagePullPolicy": "IfNotPresent",
                                "name": "mean-classifier",
                                "resources": {
                                    "requests": {
                                        "memory": "1Mi"
                                    }
                                }
                            }
                        ],
                        "terminationGracePeriodSeconds": 20
                    }
                },
                "graph": {
                    "children": [],
                    "name": "mean-classifier",
                    "endpoint": {
			"type" : "REST"
		    },
                    "type": "MODEL"
                },
                "name": "fx-market-predictor",
                "replicas": 1,
		"annotations": {
		    "predictor_version" : "v1"
		}
            }
        ]
    }
}
```