# k8s-debug - LLM-Powered Kubernetes Error Analyzer

A command-line tool that uses LLM (Ollama) to analyze and explain Kubernetes errors piped in via stdin.

## Features

- Read Kubernetes errors/logs from stdin
- Send to Ollama LLM for intelligent analysis
- Three analysis modes: quick, explain, debug
- Configurable Ollama endpoint and model
- Connection testing
- Verbose logging for troubleshooting

## Installation

### 1. Install Python dependencies

```bash
# Create and activate virtual environment (recommended)
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### 2. Make the script executable

```bash
chmod +x k8s-debug
```

### 3. (Optional) Add to PATH

```bash
# Add to ~/.bashrc or ~/.zshrc
export PATH="$PATH:/path/to/k8s-inference/cmdline-tools"

# Or create a symlink
sudo ln -s /path/to/k8s-inference/cmdline-tools/k8s-debug /usr/local/bin/k8s-debug
```

## Prerequisites

The tool is configured to use `http://llm.apexkube.xyz` as the default Ollama endpoint.

Make sure:
- The LLM service at `http://llm.apexkube.xyz` is running and accessible from your network
- You can reach the endpoint (test with: `curl http://llm.apexkube.xyz/`)

### Using a Different Endpoint
If you need to use a different Ollama instance, use the `--url` parameter:
```bash
# Use local port-forwarded service
kubectl port-forward -n monitoring svc/ollama 11434:11434
kubectl logs my-pod | k8s-debug --url http://localhost:11434
```

## Usage

### Basic Usage

```bash
# Analyze pod logs
kubectl logs my-failing-pod | k8s-debug

# Analyze pod description
kubectl describe pod my-pod | k8s-debug

# Analyze from a file
cat error.log | k8s-debug
k8s-debug < error.log
```

### Analysis Modes

#### Quick Mode (2-3 sentences)
```bash
kubectl logs my-pod | k8s-debug --mode quick
```

#### Explain Mode (default - detailed with fixes)
```bash
kubectl logs my-pod | k8s-debug --mode explain
# or just
kubectl logs my-pod | k8s-debug
```

#### Debug Mode (comprehensive debugging guide)
```bash
kubectl logs my-pod | k8s-debug --mode debug
```

### Custom Configuration

#### Different Ollama URL
```bash
kubectl logs my-pod | k8s-debug --url http://ollama.example.com:11434
```

#### Different Model
```bash
kubectl logs my-pod | k8s-debug --model llama3.2:3b
```

#### Verbose Output
```bash
kubectl logs my-pod | k8s-debug -v
```

### Connection Testing

```bash
k8s-debug --test
```

## Examples

### Example 1: CrashLoopBackOff Error

```bash
kubectl describe pod nginx-pod | k8s-debug
```

Output:
```
Analyzing with LLM...

The pod is experiencing a CrashLoopBackOff error, which means the container is
repeatedly crashing and Kubernetes is backing off on restart attempts.

Most likely cause: The application inside the container is failing to start or
is crashing immediately after startup.

Steps to fix:
1. Check the container logs: kubectl logs nginx-pod
2. Look for application errors, missing dependencies, or configuration issues
3. Verify the container image is correct and not corrupted
4. Check if required environment variables or config maps are properly set
5. Ensure the container's startup command/entrypoint is valid
```

### Example 2: ImagePullBackOff

```bash
kubectl get events | grep ImagePullBackOff | k8s-debug --mode quick
```

### Example 3: OOMKilled

```bash
kubectl get pod my-pod -o yaml | k8s-debug --mode debug
```

## Practical Workflows

### 1. Quick Pod Debugging
```bash
# One-liner to debug a failing pod
kubectl logs failing-pod --tail=100 | k8s-debug
```

### 2. Event Analysis
```bash
# Analyze recent cluster events
kubectl get events --all-namespaces | grep -i error | k8s-debug
```

### 3. Multi-pod Analysis
```bash
# Debug all pods with issues in a namespace
for pod in $(kubectl get pods -n my-namespace --field-selector=status.phase!=Running -o name); do
  echo "=== Analyzing $pod ==="
  kubectl describe $pod -n my-namespace | k8s-debug --mode quick
  echo ""
done
```

### 4. Application Log Analysis
```bash
# Analyze application errors
kubectl logs my-app --since=1h | grep ERROR | k8s-debug
```

## Command-Line Options

```
usage: k8s-debug [-h] [--url URL] [--model MODEL] [--mode {quick,explain,debug}]
                 [-v] [--test]

Options:
  -h, --help            Show this help message and exit
  --url URL             Ollama API URL (default: http://llm.apexkube.xyz)
  --model MODEL         Model to use (default: llama3.2:1b)
  --mode {quick,explain,debug}
                        Analysis mode (default: explain)
  -v, --verbose         Enable verbose output
  --test                Test connection to Ollama and exit
```

## Troubleshooting

### Connection Issues

```bash
# Test the connection to the default endpoint
k8s-debug --test

# Check if the LLM service is accessible
curl http://llm.apexkube.xyz/

# If using a different endpoint, test with that URL
k8s-debug --url http://localhost:11434 --test
```

### Timeout Issues

If you're getting timeouts with large inputs:
- The LLM might be processing a large amount of data
- Try filtering/reducing the input: `kubectl logs pod | tail -100 | k8s-debug`
- Consider using a more powerful model

### Model Not Found

```bash
# List available models
curl http://localhost:11434/api/tags

# Pull the model if needed (on the k8s cluster)
kubectl exec -n monitoring deployment/ollama -- ollama pull llama3.2:1b
```

## Integration with Shell Aliases

Add to your `.bashrc` or `.zshrc`:

```bash
# Quick k8s debugging
alias kdebug='k8s-debug'
alias kdebug-quick='k8s-debug --mode quick'
alias kdebug-full='k8s-debug --mode debug'

# Analyze last pod logs
klog-debug() {
    kubectl logs "$1" --tail=100 | k8s-debug
}

# Describe and debug
kdesc-debug() {
    kubectl describe pod "$1" | k8s-debug
}
```

Usage:
```bash
klog-debug my-failing-pod
kdesc-debug my-pod
```

## Performance Tips

1. **Filter input**: Don't pipe entire massive log files
   ```bash
   kubectl logs big-pod | tail -500 | k8s-debug
   ```

2. **Use quick mode** for faster responses
   ```bash
   kubectl logs pod | k8s-debug --mode quick
   ```

3. **Target specific errors**
   ```bash
   kubectl logs pod | grep -A 5 -B 5 ERROR | k8s-debug
   ```

## Architecture

```
┌─────────────────┐
│  kubectl/logs   │
│   (any input)   │
└────────┬────────┘
         │ (pipe)
         ▼
┌─────────────────┐
│   k8s-debug     │
│  (Python CLI)   │
└────────┬────────┘
         │ HTTP POST
         ▼
┌─────────────────┐
│  Ollama API     │
│ (llama3.2:1b)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  LLM Analysis   │
│   & Response    │
└─────────────────┘
```

## License

MIT
