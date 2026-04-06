# 🔒 Sandboxing Tool Execution

> **Phase 5 · Article 3 of 12** | ⏱️ 18 min read | 🏷️ `#sandboxing` `#isolation` `#execution-control` `#containers`

---

## TL;DR

- **Code execution tools (shell, Python eval, etc.) must NEVER run in the agent process** — they need OS-level isolation to contain compromise.
- **Sandboxing layers**: Docker containers, gVisor/Firecracker VMs, seccomp profiles, network isolation, resource limits, read-only filesystems.
- **No single sandbox is perfect** — Docker escapes exist; use layered defenses (container + seccomp + network namespace + resource cgroup).
- **Practical reality**: For most production agents, a Docker container with seccomp profile + network policy + resource limits is sufficient; for highest-trust environments, add gVisor or Firecracker.

---

## Why Tool Execution MUST Be Sandboxed

```
THE DISASTER SCENARIO (no sandbox):
─────────────────────────────────────────────────────────────────
Agent: "Run this Python code:"
  code = user_provided_code  # User submits: "import os; os.system('rm -rf /')"

Execution: eval(code) inside agent process
  ↓
Python evaluates the malicious code
  ↓
Agent process has access to: memory, file descriptors, environment,
  database credentials, API keys in memory
  ↓
Attacker gains full compromise of everything the agent can access

WITH SANDBOX (Docker container):
─────────────────────────────────────────────────────────────────
Agent: "Run this Python code:"
  code = user_provided_code

Execution: Spawn Docker container with:
  - No network access
  - /tmp read-only or ephemeral
  - 512MB memory limit
  - 5 second timeout
  - No access to host filesystem

Malicious code runs in isolated container
  ↓
Container killed after 5 seconds or memory limit hit
  ↓
Damage contained: only /tmp files lost (which are ephemeral anyway)
  ↓
Agent process unaffected, no credential access
```

---

## The Sandbox Taxonomy

### Isolation Levels

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SANDBOX ISOLATION LEVELS                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ LEVEL 1: NO SANDBOX ❌                                               │
│ ─────────────────────                                               │
│ Code runs in agent process                                          │
│ Compromise = full agent compromise                                  │
│ Use case: NEVER (except trusted, pre-written code only)            │
│                                                                       │
│ LEVEL 2: PROCESS ISOLATION (chroot, seccomp)                        │
│ ────────────────────────────────────────────                        │
│ Code runs in child process with limited syscalls                    │
│ Kernel shared; can escape via kernel exploits                       │
│ Isolation strength: Medium                                           │
│ Overhead: Low (<5%)                                                 │
│ Use case: Trusted code, internal-only agents                        │
│                                                                       │
│ LEVEL 3: CONTAINER ISOLATION (Docker)                               │
│ ──────────────────────────────────────                              │
│ Code runs in Linux container (cgroups + namespaces)                │
│ Kernel shared; sandbox escape research ongoing                      │
│ Isolation strength: High (with proper config)                       │
│ Overhead: Medium (10-20%)                                           │
│ Use case: Most production agents                                    │
│                                                                       │
│ LEVEL 4: VM ISOLATION (gVisor, Firecracker)                        │
│ ─────────────────────────────────────────                           │
│ Code runs in user-space OS (gVisor) or lightweight VM               │
│ Kernel not shared; sandbox escape requires kernel-like impl bug    │
│ Isolation strength: Very high                                       │
│ Overhead: High (30-50%)                                             │
│ Use case: Untrusted user code, high-security environments          │
│                                                                       │
│ LEVEL 5: FULL VM ISOLATION (QEMU, KVM)                             │
│ ──────────────────────────────────────                              │
│ Code runs in full virtual machine                                   │
│ Complete isolation; can only attack hypervisor                      │
│ Isolation strength: Extreme (but hypervisor vulns exist)           │
│ Overhead: Very high (50%+)                                          │
│ Use case: Untrusted code evaluation platforms                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Layer 3A: Docker Sandboxing

### Docker Security Configuration

The most practical approach for most agents is Docker with security hardening:

```python
# ❌ WRONG: Running code execution tool in agent process
def execute_python_code(code: str) -> str:
    """DANGEROUS — no isolation"""
    try:
        result = eval(code)  # Code has access to agent memory
        return str(result)
    except Exception as e:
        return f"Error: {e}"

# ✅ RIGHT: Docker-sandboxed execution
import docker
import json
import tempfile
from dataclasses import dataclass
from typing import Optional

@dataclass
class SandboxExecResult:
    stdout: str
    stderr: str
    exit_code: int
    timed_out: bool
    exceeded_memory: bool

class DockerSandbox:
    """
    Sandboxed code execution using Docker containers.
    Each execution gets: 512MB memory, 5 second timeout, no network.
    """

    def __init__(self, image: str = "python:3.11-slim",
                 memory_mb: int = 512,
                 timeout_seconds: int = 5):
        self.client = docker.from_env()
        self.image = image
        self.memory_mb = memory_mb
        self.timeout_seconds = timeout_seconds
        self._ensure_image()

    def _ensure_image(self):
        """Pull image if not present"""
        try:
            self.client.images.get(self.image)
        except docker.errors.ImageNotFound:
            print(f"Pulling {self.image}...")
            self.client.images.pull(self.image)

    def execute(self, code: str, timeout_override: Optional[int] = None) -> SandboxExecResult:
        """
        Execute code in isolated container.

        Security properties:
          - No network access (--network none)
          - Memory limited (--memory 512m)
          - CPU limited (--cpus 1)
          - Filesystem isolation (container filesystem only)
          - PID 1 only (no fork bombs)
          - Readonly root except /tmp
          - No privileged capabilities
        """
        timeout = timeout_override or self.timeout_seconds

        try:
            # Container runs as non-root, no extended capabilities
            container = self.client.containers.run(
                self.image,
                command=["python", "-c", code],

                # Resource limits
                mem_limit=f"{self.memory_mb}m",
                memswap_limit=f"{self.memory_mb}m",  # Prevent swap escape
                cpus=1.0,  # Single CPU core

                # Network isolation
                network_mode="none",  # No network access at all

                # Filesystem isolation
                read_only=False,  # Container has /tmp
                tmpfs={'/tmp': 'size=10m,noexec'},  # Small temp space

                # User/capability isolation
                user="nobody:nobody",
                cap_drop=["ALL"],  # Drop all capabilities

                # PID isolation
                ipc_mode="private",  # No IPC with host
                pid_mode="container",  # PID namespace

                # Runtime limits
                timeout=timeout,
                detach=False,

                # Name for tracking
                name=f"agent-exec-{int(time.time()*1000)}",
            ) as result:
                return SandboxExecResult(
                    stdout=result.stdout.decode() if result.stdout else "",
                    stderr=result.stderr.decode() if result.stderr else "",
                    exit_code=result.exit_code,
                    timed_out=False,
                    exceeded_memory=False
                )

        except docker.errors.APIError as e:
            if "OOMKilled" in str(e):
                return SandboxExecResult(
                    stdout="",
                    stderr="Memory limit exceeded",
                    exit_code=137,
                    timed_out=False,
                    exceeded_memory=True
                )
            elif "Timeout" in str(e) or timeout in str(e):
                return SandboxExecResult(
                    stdout="",
                    stderr=f"Execution timeout ({timeout}s)",
                    exit_code=124,
                    timed_out=True,
                    exceeded_memory=False
                )
            else:
                return SandboxExecResult(
                    stdout="",
                    stderr=f"Execution error: {str(e)}",
                    exit_code=1,
                    timed_out=False,
                    exceeded_memory=False
                )

# Usage in agent
sandbox = DockerSandbox(memory_mb=512, timeout_seconds=5)

# Attacker tries to exfiltrate secrets
malicious_code = """
import os
creds = os.environ.get('AGENT_API_KEY')
# ... send to attacker
"""

result = sandbox.execute(malicious_code)
# Even if attacker had the code, they get:
# - No network to exfil (network=none)
# - 512MB memory limit (can't process large data)
# - 5 second timeout (not enough time for exfil)
# - No access to host environment (isolated)
```

### Docker Compose Security Configuration

For deployment, use docker-compose with security defaults:

```yaml
version: '3.8'
services:
  agent:
    image: agent:latest
    environment:
      # Credentials NOT in container (use external secrets)
      DATABASE_URL: "" # Injected at runtime from secrets manager

    # Resource limits
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G
        reservations:
          cpus: '1'
          memory: 512M

    # Security options
    security_opt:
      - no-new-privileges:true
      - apparmor=docker-default  # AppArmor profile
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # Only if listening on port

    # Filesystem
    read_only: true
    tmpfs:
      - /tmp
      - /run
    volumes:
      - /var/log/agent:/var/log:ro  # Logs only

    # Networking
    networks:
      - agent-network
    ports:
      - "127.0.0.1:8000:8000"  # Localhost only

  # Sandboxed execution environment (separate container)
  sandbox:
    image: python:3.11-slim
    entrypoint: /bin/sleep
    command: ["infinity"]

    environment:
      # No credentials here either

    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M

    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL

    read_only: true
    tmpfs:
      - /tmp:size=10m,noexec

    networks:
      - sandbox-network  # Separate network, no internet

networks:
  agent-network:
    driver: bridge
  sandbox-network:
    driver: bridge
```

---

## Layer 3B: Kernel-Level Isolation (seccomp + namespaces)

For additional hardening on top of containers:

```python
import subprocess
import json

class SeccompSandbox:
    """
    Layered isolation: Docker container + seccomp profile.
    Blocks dangerous syscalls even if container escapes.
    """

    # Minimal seccomp profile for Python code execution
    MINIMAL_SECCOMP_PROFILE = {
        "defaultAction": "SCMP_ACT_ERRNO",
        "defaultErrnoRet": 1,
        "archMap": [
            {
                "architecture": "SCMP_ARCH_X86_64",
                "subArchitectures": ["SCMP_ARCH_X86", "SCMP_ARCH_X32"]
            }
        ],
        "syscalls": [
            # File operations (read-only)
            {
                "names": [
                    "open", "openat", "openat2",
                    "read", "pread64", "readv", "preadv", "preadv2",
                    "close", "stat", "fstat", "lstat", "access",
                    "faccessat", "faccessat2"
                ],
                "action": "SCMP_ACT_ALLOW"
            },
            # Process operations (limited)
            {
                "names": [
                    "execve",  # Blocked for additional safety
                    "fork",    # Blocked to prevent fork bombs
                    "clone",   # Blocked
                ],
                "action": "SCMP_ACT_ERRNO"
            },
            # Memory operations
            {
                "names": [
                    "mmap", "mprotect", "munmap", "brk",
                    "madvise", "msync"
                ],
                "action": "SCMP_ACT_ALLOW"
            },
            # Signal handling
            {
                "names": [
                    "sigaction", "sigprocmask", "sigaltstack"
                ],
                "action": "SCMP_ACT_ALLOW"
            },
            # Essential syscalls
            {
                "names": [
                    "write", "writev", "pwrite64", "pwritev", "pwritev2",
                    "exit", "exit_group",
                    "rt_sigaction", "rt_sigprocmask", "rt_sigreturn",
                    "ioctl", "poll", "epoll_create", "epoll_ctl",
                    "getpid", "getppid", "getpgid", "getuid", "getgid",
                    "geteuid", "getegid", "gettid",
                    "clock_gettime", "clock_nanosleep",
                    "futex", "futex2"
                ],
                "action": "SCMP_ACT_ALLOW"
            },
            # Network operations (BLOCKED - network=none should prevent)
            {
                "names": [
                    "socket", "socketpair", "connect", "bind",
                    "listen", "accept", "sendto", "recvfrom"
                ],
                "action": "SCMP_ACT_ERRNO"
            },
            # ptrace, eBPF (BLOCKED)
            {
                "names": [
                    "ptrace", "bpf", "perf_event_open", "prctl"
                ],
                "action": "SCMP_ACT_ERRNO"
            }
        ]
    }

    @staticmethod
    def write_profile(output_path: str):
        """Write seccomp profile to file for Docker to use"""
        import json
        with open(output_path, 'w') as f:
            json.dump(SeccompSandbox.MINIMAL_SECCOMP_PROFILE, f, indent=2)
        print(f"Seccomp profile written to {output_path}")

# Usage in docker run:
# docker run --security-opt seccomp=/path/to/profile.json ...
```

---

## Layer 3C: VM-Based Sandboxing (gVisor)

For highest isolation, use gVisor instead of Docker:

```python
class GvisorSandbox:
    """
    gVisor (runsc) provides user-space OS implementation.
    Code cannot directly call host kernel → must go through gVisor.

    Trade-off: 30-50% overhead, but sandbox escape requires
    finding bug in gVisor implementation (not kernel).
    """

    def execute_with_gvisor(self, code: str) -> SandboxExecResult:
        """
        Use gVisor runtime instead of runc.

        Installation:
          curl -fsSL https://gvisor.dev/archive/latest/containerd/runsc \
            | sudo bash -s -- --latest /usr/local/bin

        Docker config (/etc/docker/daemon.json):
          {
            "runtimes": {
              "runsc": {
                "path": "/usr/local/bin/runsc",
                "runtimeArgs": [
                  "--network=none",
                  "--file-access=exclusive"
                ]
              }
            }
          }
        """
        try:
            # Run with gvisor runtime
            container = self.client.containers.run(
                self.image,
                command=["python", "-c", code],

                runtime="runsc",  # gVisor runtime

                # All other configs same as before
                mem_limit="512m",
                memswap_limit="512m",
                cpus=1.0,
                network_mode="none",
                read_only=True,
                tmpfs={'/tmp': 'size=10m,noexec'},
                user="nobody:nobody",
                cap_drop=["ALL"],

                timeout=5,
                detach=False,
            ) as result:
                return SandboxExecResult(
                    stdout=result.stdout.decode() if result.stdout else "",
                    stderr=result.stderr.decode() if result.stderr else "",
                    exit_code=result.exit_code,
                    timed_out=False,
                    exceeded_memory=False
                )

        except Exception as e:
            return SandboxExecResult(
                stdout="",
                stderr=f"gVisor execution failed: {str(e)}",
                exit_code=1,
                timed_out=False,
                exceeded_memory=False
            )

# Firecracker: Lightweight VM
# Similar to gVisor but uses KVM instead of user-space OS
# Use case: Very high isolation, thousands of parallel executions
# Overhead: Higher than gVisor, but still acceptable (20-40%)
```

---

## Comparison: Isolation Levels

```
┌──────────────┬──────────────┬────────────┬──────────────┬─────────────┐
│ Approach     │ Overhead     │ Isolation  │ Escape Risk  │ Best For    │
├──────────────┼──────────────┼────────────┼──────────────┼─────────────┤
│ No sandbox   │ 0%           │ ❌ None   │ ❌ Critical  │ NEVER       │
│ chroot+      │ 5%           │ 🟡 Low    │ 🔴 High      │ Trusted     │
│ seccomp      │              │            │              │ code only   │
├──────────────┼──────────────┼────────────┼──────────────┼─────────────┤
│ Docker       │ 10-20%       │ 🟢 High   │ 🟡 Medium    │ Most prod   │
│ (hardened)   │              │            │              │ agents      │
├──────────────┼──────────────┼────────────┼──────────────┼─────────────┤
│ Docker +     │ 15-25%       │ 🟢 High   │ 🟡 Medium    │ Prod with   │
│ seccomp +    │              │            │              │ high risk   │
│ AppArmor     │              │            │              │ tools       │
├──────────────┼──────────────┼────────────┼──────────────┼─────────────┤
│ gVisor       │ 30-50%       │ 🟢 Very   │ 🟢 Low       │ Untrusted   │
│              │              │ high       │              │ user code   │
├──────────────┼──────────────┼────────────┼──────────────┼─────────────┤
│ Firecracker/ │ 30-40%       │ 🟢 Very   │ 🟢 Very Low  │ Extreme     │
│ KVM          │              │ high       │              │ isolation   │
└──────────────┴──────────────┴────────────┴──────────────┴─────────────┘
```

---

## Complete Secure Sandbox Example

```python
import docker
import logging
import time
from typing import Optional
from dataclasses import dataclass

logger = logging.getLogger(__name__)

@dataclass
class ExecutionConfig:
    """Configurable sandbox parameters"""
    memory_mb: int = 512
    timeout_seconds: int = 5
    max_output_bytes: int = 1_000_000  # 1MB max output
    allow_filesystem: bool = False
    isolation_level: str = "docker"  # "docker", "gvisor", "firecracker"

class SecureExecutionSandbox:
    """
    Production-ready sandboxed execution environment.
    Combines multiple isolation layers for defense-in-depth.
    """

    def __init__(self, config: ExecutionConfig = None):
        self.config = config or ExecutionConfig()
        self.client = docker.from_env()
        self.execution_id_counter = 0

    def execute_safely(self, code: str, language: str = "python") -> dict:
        """
        Execute code with security layers:
        1. Resource limits (memory, CPU, timeout)
        2. Filesystem isolation (read-only root)
        3. Network isolation (no network)
        4. Process isolation (container)
        5. Capability dropping (no root)
        """
        self.execution_id_counter += 1
        exec_id = f"exec-{self.execution_id_counter}-{int(time.time())}"

        try:
            logger.info(f"[{exec_id}] Starting execution", extra={
                "code_length": len(code),
                "language": language,
                "memory_mb": self.config.memory_mb,
                "timeout_s": self.config.timeout_seconds
            })

            image = f"{language}:latest"
            cmd_map = {
                "python": ["python", "-c", code],
                "node": ["node", "-e", code],
                "bash": ["bash", "-c", code],
            }

            if language not in cmd_map:
                return {
                    "success": False,
                    "error": f"Unsupported language: {language}",
                    "execution_id": exec_id
                }

            start_time = time.time()

            try:
                container = self.client.containers.run(
                    image,
                    command=cmd_map[language],

                    # Resource isolation
                    mem_limit=f"{self.config.memory_mb}m",
                    memswap_limit=f"{self.config.memory_mb}m",
                    cpus=0.5,  # Half a CPU

                    # Network isolation
                    network_mode="none",

                    # Filesystem isolation
                    read_only=True,
                    tmpfs={'/tmp': f'size={self.config.memory_mb // 2}m,noexec'},

                    # User isolation
                    user="nobody",
                    cap_drop=["ALL"],

                    # Isolation level
                    runtime=self._get_runtime(),

                    # Execution
                    timeout=self.config.timeout_seconds,
                    detach=False,
                    name=exec_id,

                    # Cleanup
                    auto_remove=True,

                ) as result:

                    elapsed = time.time() - start_time

                    stdout = (result.stdout or b'').decode('utf-8', errors='replace')
                    stderr = (result.stderr or b'').decode('utf-8', errors='replace')

                    # Truncate large outputs
                    if len(stdout) > self.config.max_output_bytes:
                        stdout = stdout[:self.config.max_output_bytes] + "\n[OUTPUT TRUNCATED]"

                    return {
                        "success": result.exit_code == 0,
                        "exit_code": result.exit_code,
                        "stdout": stdout,
                        "stderr": stderr,
                        "elapsed_seconds": elapsed,
                        "execution_id": exec_id
                    }

            except docker.errors.APIError as e:
                elapsed = time.time() - start_time

                if "OOMKilled" in str(e):
                    return {
                        "success": False,
                        "error": f"Memory limit exceeded ({self.config.memory_mb}MB)",
                        "elapsed_seconds": elapsed,
                        "execution_id": exec_id
                    }
                elif elapsed > self.config.timeout_seconds:
                    return {
                        "success": False,
                        "error": f"Timeout exceeded ({self.config.timeout_seconds}s)",
                        "elapsed_seconds": elapsed,
                        "execution_id": exec_id
                    }
                else:
                    logger.exception(f"[{exec_id}] Docker execution error", extra={
                        "error": str(e),
                        "elapsed": elapsed
                    })
                    return {
                        "success": False,
                        "error": f"Execution error: {str(e)[:200]}",
                        "execution_id": exec_id
                    }

        except Exception as e:
            logger.exception(f"[{exec_id}] Unexpected error", extra={"error": str(e)})
            return {
                "success": False,
                "error": f"Internal error: {str(e)[:200]}",
                "execution_id": exec_id
            }

    def _get_runtime(self) -> str:
        """Get runtime based on isolation level"""
        if self.config.isolation_level == "gvisor":
            return "runsc"
        elif self.config.isolation_level == "firecracker":
            return "firecracker"
        else:
            return "runc"  # Default Docker runtime
```

---

## Sandboxing Checklist

```
SANDBOXING SECURITY CHECKLIST
────────────────────────────────────────────────────────────────

EXECUTION ISOLATION:
  ☐ Code execution tools run in separate container/VM
  ☐ Container does NOT share network namespace with agent
  ☐ Container process does NOT inherit agent environment variables
  ☐ No volume mounts to sensitive host directories
  ☐ Filesystem is read-only except for /tmp (ephemeral)

RESOURCE LIMITS:
  ☐ Memory limit enforced (512MB typical)
  ☐ Memory swap disabled (prevent swap escape)
  ☐ CPU limit enforced (0.5-1 CPU core)
  ☐ Timeout enforced (5 seconds typical)
  ☐ Max output size enforced (1MB typical)

CAPABILITY DROPPING:
  ☐ All Linux capabilities dropped (cap_drop=ALL)
  ☐ Not running as root (user="nobody")
  ☐ No privileged flag (privileged=false)
  ☐ No device access (/dev unmounted except /dev/null, /dev/urandom)

NETWORK ISOLATION:
  ☐ Network set to "none" (no network access)
  ☐ No DNS access
  ☐ Cannot call host or other containers
  ☐ If network needed: restricted to allowlist only

KERNEL-LEVEL HARDENING:
  ☐ seccomp profile applied (if using container isolation)
  ☐ AppArmor or SELinux profile applied
  ☐ no-new-privileges flag set
  ☐ Read-only root filesystem

OPERATIONAL:
  ☐ Container automatically removed after execution
  ☐ Execution logs captured (stdout, stderr, exit code)
  ☐ Timeout kills container, doesn't hang
  ☐ Memory limit kills container, doesn't corrupt host
  ☐ Monitoring: alert on frequent timeouts/OOM kills
  ☐ Rate limiting: prevent sandbox resource exhaustion

SELECTION:
  ☐ Docker + seccomp: for most production use cases
  ☐ gVisor: for untrusted user code
  ☐ Firecracker: for extreme isolation or high parallelism
  ☐ Multiple sandbox types available for different threat models
```

---

## Further Reading

- **Docker Security Best Practices**: https://docs.docker.com/engine/security/
- **seccomp in Docker**: https://docs.docker.com/engine/security/seccomp/
- **gVisor Security**: https://gvisor.dev/docs/
- **Firecracker Design**: https://firecracker-microvm.github.io/
- **Sandbox Escape Research**: Container security advisories and CVE databases
- **OWASP: Code Injection**: https://owasp.org/www-community/attacks/Code_Injection
- **Kubernetes Pod Security Standards**: https://kubernetes.io/docs/concepts/security/pod-security-standards/

---

← [Prev: Input Validation & Output Sanitization](./02-input-output-validation.md) | [Next: Human-in-the-Loop Design →](./04-human-in-the-loop.md)
