# How to make a Controller with Wrangler

## Create a Go module for your controller, add wrangler module

```shell
mkdir pcidevices
go mod init github.com/harvester/pcidevices
go get github.com/rancher/wrangler
```

To make the tutorial easier to follow, set your GVK in environment variables:

```shell
export GROUP=devices
export DOMAIN=harvesterhci.io
export VERSION=v1beta1
```

## Create your CRD as a Go Type:

```shell
mkdir -p pkg/apis/$GROUP.$DOMAIN/$VERSION/
touch pkg/apis/$GROUP.$DOMAIN/$VERSION/pcidevice.go
```

### Edit your CRD Go file and add code generation comments

This is the file that Wrangler will use to generate your controllers.
For example, I am going to make a PCIDevice type, representing a single PCI Device on a single node:

```shell
$EDITOR pkg/apis/$GROUP.$DOMAIN/$VERSION/pcidevice.go
```

```go
// +genclient
// +genclient:nonNamespaced
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +kubebuilder:object:root=true

// PCIDevice is the Schema for the pcidevices API
type PCIDevice struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   PCIDeviceSpec   `json:"spec,omitempty"`
	Status PCIDeviceStatus `json:"status,omitempty"`
}
// PCIDeviceStatus defines the observed state of PCIDevice
type PCIDeviceStatus struct {
	Address           string   `json:"address"`
	VendorId          string   `json:"vendorId"`
	DeviceId          string   `json:"deviceId"`
	Node              Node     `json:"node"`
	Description       string   `json:"description"`
	KernelDriverInUse string   `json:"kernelDriverInUse,omitempty"`
	KernelModules     []string `json:"kernelModules"`
}

type PCIDeviceSpec struct {
}

type Node struct {
	SystemUUID string `json:"systemUUID"`
	Name       string `json:"name"`
}

```


## Create your boilerplate
    
```shell
mkdir scripts
cat <<EOF > scripts/boilerplate.go.txt
/*
Copyright $(date +%Y) Rancher Labs, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/
EOF
```


## Create your codegen main.go

Import Wrangler's controllergen

```shell
mkdir -p pkg/codegen/
touch pkg/codegen/main.go
$EDITOR pkg/codegen/main.go
```

And fill out the main.go 

```go
package main

import (
    "fmt"
    "os"
    controllergen "github.com/rancher/wrangler/pkg/controller-gen"
    "github.com/rancher/wrangler/pkg/controller-gen/args"
	// Ensure gvk gets loaded in wrangler/pkg/gvk cache
	_ "github.com/rancher/wrangler/pkg/generated/controllers/apiextensions.k8s.io/v1"
)

func main() {
    os.Unsetenv("GOPATH")
    controllergen.Run(
        args.Options{
            OutputPackage: "github.com/harvester/pcidevices",
            Boilerplate: "scripts/boilerplate.go.txt",
            Groups: map[string]args.Group{
                "devices.harvesterhci.io": {
                    Types: []any{
                        "./pkg/apis/devices.harvesterhci.io/v1beta1",
                    },
                    
                }
            }
        }
    )
}
```


## Create your codegen cleanup main.go

```shell
mkdir -p pkg/codegen/cleanup
touch pkg/codegen/cleanup/main.go
$EDITOR pkg/codegen/cleanup/main.go
```


```go
package main

import (
	"os"

	"github.com/rancher/wrangler/pkg/cleanup"
	"github.com/sirupsen/logrus"
)

func main() {
	if err := cleanup.Cleanup("./pkg/apis"); err != nil {
		logrus.Fatal(err)
	}
	if err := os.RemoveAll("./pkg/generated"); err != nil {
		logrus.Fatal(err)
	}
}
```


## Create generate.go 

```shell
touch generate.go
$EDITOR generate.go
```

```go
//go:generate go run pkg/codegen/cleanup/main.go
//go:generate go run pkg/codegen/main.go

package main
```

## Run `go generate`

Now the fun part, run `go generate` and inspect the generated files, 
there are some under a new top-level directory called `controllers/` 
and more under `pkg/`.

```shell
go generate
```
    
