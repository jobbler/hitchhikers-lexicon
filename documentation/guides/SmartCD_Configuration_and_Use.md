
# SmartCD Configuration and Use

| **Author:** | John Herr          |
| :---------- | ------------------ |
| **Date:**   | Novemeber 20, 2025 |

> **⚠️ Editor's Note/Disclaimer:** This document was collaboratively reviewed and refined by an expert AI (Gemini) to enhance its clarity, structure, and readability for publication. The core technical content, expertise, and original configuration details were provided by the author.


## Overview
SmartCD is a tool for managing directory-specific shell configurations, enabling users to automate environment setup and cleanup tasks when entering or exiting directories.

---

## Getting Started

### Download
You can obtain SmartCD from the official or forked repository:
- [Official GitHub Repository](https://github.com/cxreg/smartcd)
- [Forked Version](https://github.com/jobbler/forked-smartcd)

Choose to either clone the repository or download and extract the archive.

---

## Installation

To install SmartCD, follow these steps:

```bash
make install
source load_smartcd
smartcd config
```

Ensure the `load_smartcd` script is sourced in your shell environment for SmartCD to function correctly.

---

## Configuration

SmartCD manages configurations in the `~/.smartcd/scripts/$PWD` directory. Use the following commands to edit scripts:

- `smartcd edit enter`: Edit commands to run when entering the directory.
- `smartcd edit leave`: Edit commands to run when exiting the directory.

These scripts are automatically generated and stored in the specified directory.

---

## Templates

Templates allow you to reuse configuration logic across multiple directories. SmartCD supports two methods of using templates:

### Method 1: Install Template
Install a template into a specific directory:

```bash
smartcd template install template_name
```

### Method 2: Reference Template in Scripts
Include template logic directly in your `enter` or `leave` scripts:

```bash
smartcd template run cluster
```

This approach allows you to update templates centrally, and changes will propagate to all scripts that reference them.

### Template Structure
A template contains both `enter` and `leave` logic. SmartCD automatically applies the correct portion to each script. Here's an example template:

```bash
########################################################################
# SmartCD Template: cluster
# This template creates cluster-specific directories and configurations.
########################################################################
# Enter commands
########################################################################
# smartcd enter - __PATH__
#
# Create bin, auth, and bash_completion directories if needed
[[ ! -d __PATH__/bin ]]  && mkdir -v __PATH__/bin  
[[ ! -d __PATH__/auth ]] && mkdir -v __PATH__/auth  
[[ ! -d __PATH__/.bash_completion.d ]] && mkdir -v __PATH__/.bash_completion.d  
# Source bash completion files if they exist  
for file in __PATH__/.bash_completion.d/*  
do  
  echo source $file  
done  
autostash PATH=__PATH__/bin:$PATH  
autostash KUBECONFIG=__PATH__/auth/kubeconfig  
smartcd helper run history localize __PATH__/.bash_history
########################################################################
# Leave commands
########################################################################
oc_completion=$( sed -n -e "s/^\(__.*\)().*/\1/p" __PATH__/bin/oc_bash_complete  | sort -u )
for i in $oc_completion
do
  unset -f $i
done
unset oc_completion
```

> **Note**: `__PATH__` is a placeholder that SmartCD replaces with the actual directory path when the template is applied.

---

## Advanced Features

### Enhanced Bash Array Handling
SmartCD extends bash's built-in array functionality, enabling more flexible and powerful scripting in your configuration files.

---

## Cluster-Specific Configuration

Use the following configuration for cluster environments:

### Enter Script
```bash
# Create bin, auth, and bash_completion directories if needed  
[[ ! -d __PATH__/bin ]]  && mkdir -v __PATH__/bin  
[[ ! -d __PATH__/auth ]] && mkdir -v __PATH__/auth  
[[ ! -d __PATH__/.bash_completion.d ]] && mkdir -v __PATH__/.bash_completion.d  
# Source bash completion files if they exist  
for file in __PATH__/.bash_completion.d/*  
do  
  echo source $file  
done  
autostash PATH=__PATH__/bin:$PATH  
autostash KUBECONFIG=__PATH__/auth/kubeconfig  
smartcd helper run history localize __PATH__/.bash_history
```

### Leave Script
```bash
oc_completion=$( sed -n -e "s/^\(__.*\)().*/\1/p" __PATH__/bin/oc_bash_complete  | sort -u )
for i in $oc_completion
do
  unset -f $i
done
unset oc_completion
```

---

## Additional Resources
For more information on advanced usage, see the [SmartCD website](http://smartcd.org).