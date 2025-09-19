# KCP Integration Guide

This document outlines the steps to integrate KCP (Kubernetes Cluster Platform) with the multicluster controller.

## Prerequisites

1. KCP Server installed and running
   ```bash
   # Install KCP binary
   curl -Lo kcp https://github.com/kcp-dev/kcp/releases/download/v0.28.1/kcp_v0.28.1_darwin_arm64.tar.gz
   tar -xzf kcp_v0.28.1_darwin_arm64.tar.gz
   sudo mv kcp /usr/local/bin/
   chmod +x /usr/local/bin/kcp
   
   # Start KCP server
   kcp start
   ```

2. Export KCP kubeconfig
   ```bash
   export KUBECONFIG=~/.kcp/admin.kubeconfig
   ```

## Integration Steps

### Step 1: Update go.mod with KCP Dependencies

Add KCP dependencies to your `go.mod`:

```go
require (
    github.com/kcp-dev/kcp/sdk v0.28.1
    github.com/kcp-dev/multicluster-provider v0.2.0
    // ... existing dependencies
)

// Force compatible Kubernetes versions for KCP integration
replace (
    k8s.io/api => k8s.io/api v0.32.0
    k8s.io/apiextensions-apiserver => k8s.io/apiextensions-apiserver v0.32.0
    k8s.io/apimachinery => k8s.io/apimachinery v0.32.0
    k8s.io/apiserver => k8s.io/apiserver v0.32.0
    k8s.io/client-go => k8s.io/client-go v0.32.0
    k8s.io/component-base => k8s.io/component-base v0.32.0
    sigs.k8s.io/controller-runtime => sigs.k8s.io/controller-runtime v0.19.0
)
```

### Step 2: Update main.go with KCP Integration

```go
package main

import (
    "context"
    "flag"
    "os"

    // KCP imports
    kcpapiv1alpha2 "github.com/kcp-dev/kcp/sdk/apis/tenancy/v1alpha2"
    "github.com/kcp-dev/multicluster-provider/pkg/apiexportprovider"
    
    // Standard imports
    "k8s.io/apimachinery/pkg/runtime"
    utilruntime "k8s.io/apimachinery/pkg/util/runtime"
    clientgoscheme "k8s.io/client-go/kubernetes/scheme"
    _ "k8s.io/client-go/plugin/pkg/client/auth"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/healthz"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"
    mcmanager "sigs.k8s.io/multicluster-runtime/pkg/manager"

    appv1alpha1 "github.com/WesleyDevops91/KCP-Demo-Project/api/v1alpha1"
    "github.com/WesleyDevops91/KCP-Demo-Project/internal/controller"
)

var (
    scheme   = runtime.NewScheme()
    setupLog = ctrl.Log.WithName("setup")
)

func init() {
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))
    utilruntime.Must(appv1alpha1.AddToScheme(scheme))
    utilruntime.Must(kcpapiv1alpha2.AddToScheme(scheme)) // Add KCP scheme
}

func main() {
    var metricsAddr string
    var enableLeaderElection bool
    var probeAddr string
    var secureMetrics bool
    var enableHTTP2 bool
    
    flag.StringVar(&metricsAddr, "metrics-bind-address", "0", "The address the metrics endpoint binds to. Use :8443 for HTTPS or :8080 for HTTP, or leave as 0 to disable the metrics service.")
    flag.StringVar(&probeAddr, "health-probe-bind-address", ":8081", "The address the probe endpoint binds to.")
    flag.BoolVar(&enableLeaderElection, "leader-elect", false, "Enable leader election for controller manager.")
    flag.BoolVar(&secureMetrics, "metrics-secure", true, "If set, the metrics endpoint is served securely via HTTPS.")
    flag.BoolVar(&enableHTTP2, "enable-http2", false, "If set, HTTP/2 will be enabled for the metrics and webhook servers")
    
    opts := zap.Options{
        Development: true,
    }
    opts.BindFlags(flag.CommandLine)
    flag.Parse()

    ctrl.SetLogger(zap.New(zap.UseFlagOptions(&opts)))

    // Initialize multicluster-provider for KCP integration
    provider, err := apiexportprovider.New(context.Background(), scheme)
    if err != nil {
        setupLog.Error(err, "unable to create multicluster provider")
        os.Exit(1)
    }

    // Create multicluster manager with KCP provider
    mgr, err := mcmanager.New(ctrl.GetConfigOrDie(), mcmanager.Options{
        Scheme:                 scheme,
        MetricsBindAddress:     metricsAddr,
        HealthProbeBindAddress: probeAddr,
        LeaderElection:         enableLeaderElection,
        LeaderElectionID:       "8bcb52da.example.com",
        Provider:               provider, // Use KCP provider
    })
    if err != nil {
        setupLog.Error(err, "unable to start manager")
        os.Exit(1)
    }

    if err = (&controller.SwacdappReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr); err != nil {
        setupLog.Error(err, "unable to create controller", "controller", "Swacdapp")
        os.Exit(1)
    }

    if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil {
        setupLog.Error(err, "unable to set up health check")
        os.Exit(1)
    }
    if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil {
        setupLog.Error(err, "unable to set up ready check")
        os.Exit(1)
    }

    setupLog.Info("starting manager")
    
    // Start the provider in the background
    go func() {
        if err := provider.Run(context.Background()); err != nil {
            setupLog.Error(err, "problem running provider")
        }
    }()

    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        setupLog.Error(err, "problem running manager")
        os.Exit(1)
    }
}
```

### Step 3: Update controller.go for Multicluster Awareness

```go
package controller

import (
    "context"

    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"

    appv1alpha1 "github.com/WesleyDevops91/KCP-Demo-Project/api/v1alpha1"
)

// SwacdappReconciler reconciles a Swacdapp object
type SwacdappReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

//+kubebuilder:rbac:groups=app.example.com,resources=swacdapps,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=app.example.com,resources=swacdapps/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=app.example.com,resources=swacdapps/finalizers,verbs=update

func (r *SwacdappReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    // Fetch the Swacdapp instance
    var swacdapp appv1alpha1.Swacdapp
    if err := r.Get(ctx, req.NamespacedName, &swacdapp); err != nil {
        log.Error(err, "unable to fetch Swacdapp")
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Log multicluster-aware operation
    log.Info("Reconciling Swacdapp in multicluster environment", 
             "name", swacdapp.Name, 
             "namespace", swacdapp.Namespace,
             "cluster", req.ClusterName) // This shows cluster awareness

    // TODO: Add your multicluster reconciliation logic here
    // For example:
    // - Deploy to multiple clusters based on placement policy
    // - Sync status across clusters
    // - Handle cluster-specific configurations

    return ctrl.Result{}, nil
}

// SetupWithManager sets up the controller with the Manager
func (r *SwacdappReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&appv1alpha1.Swacdapp{}).
        Complete(r)
}
```

### Step 4: Update controller_test.go

Add cluster awareness to your tests:

```go
package controller

import (
    "context"
    "path/filepath"
    "testing"

    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    
    "k8s.io/client-go/kubernetes/scheme"
    "k8s.io/client-go/rest"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/envtest"
    logf "sigs.k8s.io/controller-runtime/pkg/log"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"
    mcmanager "sigs.k8s.io/multicluster-runtime/pkg/manager"

    appv1alpha1 "github.com/WesleyDevops91/KCP-Demo-Project/api/v1alpha1"
)

var cfg *rest.Config
var k8sClient client.Client
var testEnv *envtest.Environment
var ctx context.Context
var cancel context.CancelFunc

func TestAPIs(t *testing.T) {
    RegisterFailHandler(Fail)
    RunSpecs(t, "Controller Suite")
}

var _ = BeforeSuite(func() {
    logf.SetLogger(zap.New(zap.WriteTo(GinkgoWriter), zap.UseDevMode(true)))

    ctx, cancel = context.WithCancel(context.Background())

    By("bootstrapping test environment")
    testEnv = &envtest.Environment{
        CRDDirectoryPaths:     []string{filepath.Join("..", "..", "config", "crd", "bases")},
        ErrorIfCRDPathMissing: true,
    }

    var err error
    cfg, err = testEnv.Start()
    Expect(err).NotTo(HaveOccurred())
    Expect(cfg).NotTo(BeNil())

    err = appv1alpha1.AddToScheme(scheme.Scheme)
    Expect(err).NotTo(HaveOccurred())

    // Use multicluster-runtime manager for tests (without KCP provider for testing)
    mgr, err := mcmanager.New(cfg, mcmanager.Options{
        Scheme: scheme.Scheme,
        Provider: nil, // No provider for unit tests
    })
    Expect(err).ToNot(HaveOccurred())

    err = (&SwacdappReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr)
    Expect(err).ToNot(HaveOccurred())

    go func() {
        defer GinkgoRecover()
        err = mgr.Start(ctx)
        Expect(err).ToNot(HaveOccurred(), "failed to run manager")
    }()

    k8sClient = mgr.GetClient()
    Expect(k8sClient).ToNot(BeNil())
})

var _ = AfterSuite(func() {
    By("tearing down the test environment")
    cancel()
    err := testEnv.Stop()
    Expect(err).NotTo(HaveOccurred())
})
```

## Known Issues

1. **Dependency Compatibility**: KCP requires specific Kubernetes API versions (v0.32.x) that may conflict with newer controller-runtime versions
2. **Replace Directives**: The replace directives in go.mod are necessary to force compatible versions
3. **Testing**: KCP integration should be tested separately from unit tests

## Alternative Approach: Conditional KCP Integration

For production environments where KCP may not be available, consider implementing conditional KCP integration:

```go
// In main.go
var provider mcmanager.Provider
if kcpEnabled := os.Getenv("KCP_ENABLED"); kcpEnabled == "true" {
    var err error
    provider, err = apiexportprovider.New(context.Background(), scheme)
    if err != nil {
        setupLog.Error(err, "unable to create KCP provider")
        os.Exit(1)
    }
    
    go func() {
        if err := provider.Run(context.Background()); err != nil {
            setupLog.Error(err, "problem running KCP provider")
        }
    }()
}

// Create multicluster manager with optional KCP provider
mgr, err := mcmanager.New(ctrl.GetConfigOrDie(), mcmanager.Options{
    // ... other options
    Provider: provider, // Will be nil if KCP is disabled
})
```

This allows the controller to work both with and without KCP integration.

## Verification

1. Start KCP server
2. Set KCP kubeconfig
3. Run the controller with KCP_ENABLED=true
4. Verify APIBinding resources are being watched
5. Test multicluster deployment scenarios

## Next Steps

1. Test with actual KCP workspaces
2. Implement cluster placement policies
3. Add cross-cluster resource synchronization
4. Create comprehensive integration tests