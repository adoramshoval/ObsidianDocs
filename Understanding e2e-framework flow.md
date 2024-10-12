## Goal
- [ ] Easy to maintain and understand testing repository
	- [ ] Usage of already known structures - JSON and YAML - inside `testdata` directory
	- [ ] [Table-Driven tests](https://go.dev/wiki/TableDrivenTests)
	- [ ] Ability to work with every API resource using a single client

--> Create a new Environment
	--> The Environment type is an interface, therefore, the `testEnv` works here, as it creates the different functions the Environment interface requires.
```
~package env - sigs.k8s.io/e2e-framework/pkg/env - env.go

type (
	Environment = types.Environment
	Func        = types.EnvFunc
	FeatureFunc = types.FeatureEnvFunc
	TestFunc    = types.TestEnvFunc
)

type testEnv struct {
	ctx     context.Context
	cfg     *envconf.Config
	actions []action
}

// New creates a test environment with no config attached.
func New() types.Environment {
	return newTestEnv()
}
...
...
func newTestEnv() *testEnv {
	return &testEnv{
		ctx: context.Background(),
		cfg: envconf.New(),
	}
}
```
--> `envconf.New()`
--> Here, you create a configuration type for your environment. This config will hold, among other things, a client to interact with the Kubernetes cluster, a kubeconfig as a file path and more.
```
~package envconf - sigs.k8s.io/e2e-framework/pkg/envconf - config.go

// Config represents and environment configuration
type Config struct {
	client                  klient.Client
	kubeconfig              string
	namespace               string
	assessmentRegex         *regexp.Regexp
	featureRegex            *regexp.Regexp
	labels                  flags.LabelsMap
	skipFeatureRegex        *regexp.Regexp
	skipLabels              flags.LabelsMap
	skipAssessmentRegex     *regexp.Regexp
	parallelTests           bool
	dryRun                  bool
	failFast                bool
	disableGracefulTeardown bool
	kubeContext             string
}

// New creates and initializes an empty environment configuration
func New() *Config {
	return &Config{}
}
...
...
// NewClient is a constructor function that returns a previously
// created klient.Client or create a new one based on configuration
// previously set. Will return an error if unable to do so.
func (c *Config) NewClient() (klient.Client, error) {
	if c.client != nil {
		return c.client, nil
	}

	client, err := klient.NewWithKubeConfigFile(c.kubeconfig)
	if err != nil {
		return nil, fmt.Errorf("client failed: %w", err)
	}

	return client, nil
}
```
--> `NewClient()` calls `klient.NewWithKubeConfigFile(c.kubeconfig)`, our next step.
--> Here, the `kclient` package is used in order to create a Client interface and a client struct which implements the interface's functions. 
```
~package kclient - sigs.k8s.io/e2e-framework/kclient - client.go

// Client stores values to interact with the
// API-server.
type Client interface {
	// RESTConfig returns the *rest.Config associated with this client.
	RESTConfig() *rest.Config
	// Resources returns a *Resources type to access resource CRUD operations.
	// This method takes zero or at most 1 namespace (more will panic) that
	// can be used in List operations.
	Resources(...string) *resources.Resources
}

type client struct {
	cfg       *rest.Config
	resources *resources.Resources
}

...
...

// New returns a new Client value
func New(cfg *rest.Config) (Client, error) {
	res, err := resources.New(cfg)
	if err != nil {
		return nil, err
	}
	return &client{cfg: cfg, resources: res}, nil
}

// NewWithKubeConfigFile creates a client using the kubeconfig filePath
func NewWithKubeConfigFile(filePath string) (Client, error) {
	cfg, err := conf.New(filePath)
	if err != nil {
		return nil, err
	}
	return New(cfg)
}
```
--> `NewWithKubeConfigFile` calls `conf.New` in order to create a new `*rest.Config` which is than used to interact with the cluster.
```
// New returns Kubernetes configuration value of type *rest.Config.
// filename is kubeconfig file
func New(fileName string) (*rest.Config, error) {
	var resolvedKubeConfigFile string
	kubeContext := ResolveClusterContext()

	// resolve the kubeconfig file
	resolvedKubeConfigFile = fileName
	if fileName == "" {
		resolvedKubeConfigFile = ResolveKubeConfigFile()
	}

	// if resolvedKubeConfigFile is still empty, assume in-cluster config
	if resolvedKubeConfigFile == "" {
		if kubeContext == "" {
			return rest.InClusterConfig()
		}
		// if in-cluster can't use the --kubeContext flag
		return nil, errors.New("cannot use a cluster context without a valid kubeconfig file")
	}

	// set the desired context if provided
	if kubeContext != "" {
		return NewWithContextName(resolvedKubeConfigFile, kubeContext)
	}

	// create the config object from resolvedKubeConfigFile without a context
	return clientcmd.NewNonInteractiveDeferredLoadingClientConfig(
		&clientcmd.ClientConfigLoadingRules{ExplicitPath: resolvedKubeConfigFile}, &clientcmd.ConfigOverrides{}).ClientConfig()
}

// NewWithContextName returns k8s config value of type *rest.Config
func NewWithContextName(fileName, context string) (*rest.Config, error) {
	// create the config object from k8s config path and context
	return clientcmd.NewNonInteractiveDeferredLoadingClientConfig(
		&clientcmd.ClientConfigLoadingRules{ExplicitPath: fileName},
		&clientcmd.ConfigOverrides{
			CurrentContext: context,
		}).ClientConfig()
}

// NewInCluster for clients that expect to be
// running inside a pod on kubernetes
func NewInCluster() (*rest.Config, error) {
	return rest.InClusterConfig()
}

// ResolveKubeConfigFile returns the kubeconfig file from
// either flag --kubeconfig or env KUBECONFIG.
// If flag.Parsed() is true then lookup for --kubeconfig flag.
// If --kubeconfig, or KUBECONFIG, or  $HOME/.kube/config not provided then
// assume in cluster.
func ResolveKubeConfigFile() string {
	var kubeConfigPath string

	// If a flag --kubeconfig  is specified with the config location, use that
	if flag.Parsed() {
		f := flag.Lookup("kubeconfig")
		if f != nil && f.Value.String() != "" {
			return f.Value.String()
		}
	}

	// if KUBECONFIG env is defined then use that
	kubeConfigPath = os.Getenv(clientcmd.RecommendedConfigPathEnvVar)
	if kubeConfigPath != "" {
		return kubeConfigPath
	}

	var (
		homeDir string
		ok      bool
	)

	// check if $HOME/.kube/config is present
	// if $HOME is unset, get it from current user
	if homeDir, ok = os.LookupEnv("HOME"); !ok {
		u, err := user.Current()
		if err != nil {
			// consider it as in-cluster-config
			return ""
		}

		kubeConfigPath = path.Join(u.HomeDir, clientcmd.RecommendedHomeDir, clientcmd.RecommendedFileName)
	} else {
		kubeConfigPath = path.Join(homeDir, clientcmd.RecommendedHomeDir, clientcmd.RecommendedFileName)
	}

	// check if the config path exists
	if fileExists(kubeConfigPath) {
		return kubeConfigPath
	}

	return ""
}
```

#### Summary
```
// Will create a new kclient.Client which contains both *rest.Config to interact // with the cluster and a *resources.Resources which can be used to interact with // the API and the different resources doing CRUD operations.
client, err := config.NewClient()
```

### Schemes

#### The Problem
The `k8s.io/api/core/v1` module has definitions for core Kubernetes API resources, pods, service accounts, deployments and more, in its scheme.
Therefore, the default Kubernetes RestAPI client knows how to interact with these resources natively, serialize and deserialize them.

In order to allow interaction with more CustomResourceDefinitions operators are providing, it is needed to add these resources to the scheme.
#### Goal
- [ ] Ability to work with every API resource using a single client
#### Understanding the Scheme and `AddToScheme`
- In Kubernetes, the **scheme** is used to map Go types to Kubernetes API objects and vice versa. When working with custom resources (like `CronTab`), you need to register the custom types with the scheme so the client knows how to serialize/deserialize them.
- The `AddToScheme` function registers a custom resource type with the provided scheme, allowing the client to work with it.
##### Example:

Lets say we want to list all VirtualMachineInstances in a certain namespace, "core". For that, using the `e2e-framework` Resources's type List method, like so:
```
import (
    "context"
    "testing"
    
    v1 "k8s.io/api/core/v1"
    kubev1 "kubevirt.io/api/core/v1"
    "sigs.k8s.io/e2e-framework/pkg/envconf"
    "sigs.k8s.io/e2e-framework/pkg/features"
)

func TestAPICall(t *testing.T) {
    feat := features.New("API Feature").
        WithLabel("type", "API").
        Assess("test message", func(ctx context.Context, t *testing.T, c *envconf.Config) context.Context {
            var vmis kubev1.VirtualMachineInstanceList

            if err := c.Client().Resources("core").List(ctx, &vmis); err != nil {
                t.Error(err)
            }

            t.Logf("Got VMIs %v in namespace", len(vmis.Items))
            if len(vmis.Items) == 0 {
                t.Errorf("Expected >0 VMIs in core but got %v", len(vmis.Items))
            }
            return ctx
        }).Feature()

    // testsEnvironment is the one global that we rely on; it passes the context
    // and *envconf.Config to our feature.
    testsEnvironment.Test(t, feat)
}
```

You will notice you get the following error:
```
$ no kind is registered for the type v1.VirtualMachineInstanceList in scheme "pkg/runtime/scheme.go:100"
```

Lucky for us, Kubevirt is a big enough project and they have already implemented for us the `AddToScheme` function to add each and every kind they provide to the runtime scheme.

All we have to do is adding the following line:
```
kubev1.AddToScheme(c.Client().Resources().GetScheme())
```

And the full function:
```
func TestAPICall(t *testing.T) {
    feat := features.New("API Feature").
        WithLabel("type", "API").
        Assess("test message", func(ctx context.Context, t *testing.T, c *envconf.Config) context.Context {
            var vmis kubev1.VirtualMachineInstanceList
			
			kubev1.AddToScheme(c.Client().Resources().GetScheme()) // <<<< Added line
			
            if err := c.Client().Resources("core").List(ctx, &vmis); err != nil {
                t.Error(err)
            }

            t.Logf("Got VMIs %v in namespace", len(vmis.Items))
            if len(vmis.Items) == 0 {
                t.Errorf("Expected >0 VMIs in core but got %v", len(vmis.Items))
            }
            return ctx
        }).Feature()

    // testsEnvironment is the one global that we rely on; it passes the context
    // and *envconf.Config to our feature.
    testsEnvironment.Test(t, feat)
}
```

See [CRDs example](https://github.com/kubernetes-sigs/e2e-framework/tree/517dbe336d8ca0241a8a904555e865cb0ee74549/examples/crds) from `e2e-framework` for a `AddToScheme` function to add custom CRDs to scheme.