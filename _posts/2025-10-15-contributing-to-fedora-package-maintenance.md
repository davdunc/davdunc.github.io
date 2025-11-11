---
layout: post
title: "My Journey Contributing to Fedora Package Maintenance"
date: 2025-10-15 16:45:00 -0500
categories: [linux, open-source]
tags: [fedora, rpm, packaging, open-source]
---

I recently started contributing to Fedora package maintenance, and it's been an enlightening experience. Here's what I've learned about the process and tools involved.

## Why Contribute to Package Maintenance?

Maintaining packages in a major Linux distribution teaches you:

1. The intricacies of the Linux build system
2. How to work with upstream projects
3. Collaboration in large open-source communities
4. Attention to detail in software dependencies

Plus, you're directly helping thousands of users get access to quality software.

## Getting Started

### 1. Join the Fedora Package Maintainers

First step is creating a Fedora account and signing the CLA:

```bash
# Install fedora-packager tools
sudo dnf install fedora-packager fedora-review

# Setup your FAS account
fedora-packager-setup
```

### 2. Understanding RPM Spec Files

RPM spec files are the heart of package building. Here's a simplified example:

```spec
Name:           myapp
Version:        1.0.0
Release:        1%{?dist}
Summary:        A sample application

License:        MIT
URL:            https://github.com/example/myapp
Source0:        %{url}/archive/v%{version}/%{name}-%{version}.tar.gz

BuildRequires:  gcc
BuildRequires:  make
Requires:       libawesome >= 2.0

%description
A longer description of what myapp does and why it's useful.

%prep
%autosetup

%build
%configure
%make_build

%install
%make_install

%files
%license LICENSE
%doc README.md
%{_bindir}/myapp
%{_datadir}/myapp/

%changelog
* Mon Oct 15 2025 David Duncan <email@example.com> - 1.0.0-1
- Initial package
```

### 3. Building and Testing Locally

Before submitting, always test locally:

```bash
# Install build dependencies
sudo dnf builddep myapp.spec

# Build the package
rpmbuild -ba myapp.spec

# Test install in mock (clean environment)
mock -r fedora-39-x86_64 myapp.spec
```

### 4. Using fedpkg

`fedpkg` is the tool for interacting with Fedora's build system:

```bash
# Clone a package
fedpkg clone myapp
cd myapp

# Update to new version
rpmdev-bumpspec -n 1.0.1 -c "Update to 1.0.1" myapp.spec

# Download new sources
spectool -g myapp.spec
fedpkg new-sources myapp-1.0.1.tar.gz

# Commit changes
fedpkg commit -c

# Build in Koji (Fedora's build system)
fedpkg build
```

## Real Example: Maintaining a Python Package

I maintain a Python package for data analysis. Here's the process when upstream releases a new version:

### 1. Monitor Upstream

I use a script to check for new releases:

```python
#!/usr/bin/env python3
import requests
from packaging import version

def check_pypi_version(package):
    resp = requests.get(f"https://pypi.org/pypi/{package}/json")
    latest = resp.json()["info"]["version"]
    return latest

current = "1.2.3"
latest = check_pypi_version("mypackage")

if version.parse(latest) > version.parse(current):
    print(f"New version available: {latest}")
```

### 2. Update Spec File

```spec
%global pypi_name mypackage

Name:           python-%{pypi_name}
Version:        1.2.4
Release:        1%{?dist}
Summary:        Python package for data analysis

License:        Apache-2.0
URL:            https://github.com/example/mypackage
Source0:        %{pypi_source}

BuildArch:      noarch
BuildRequires:  python3-devel
BuildRequires:  python3-setuptools

%description
Detailed description of the package.

%prep
%autosetup -n %{pypi_name}-%{version}

%generate_buildrequires
%pyproject_buildrequires -r

%build
%pyproject_wheel

%install
%pyproject_install
%pyproject_save_files %{pypi_name}

%check
%pytest

%files -f %{pyproject_files}
%license LICENSE
%doc README.md

%changelog
* Tue Oct 15 2025 David Duncan <email@example.com> - 1.2.4-1
- Update to 1.2.4
- Fixes CVE-2025-XXXXX
```

### 3. Run Package Review

```bash
# Check for common issues
rpmlint mypackage.spec

# Run fedora-review for comprehensive checks
fedora-review -n python-mypackage
```

### 4. Submit Update

```bash
# Build for all releases
fedpkg build --target f39
fedpkg build --target f40

# Create Bodhi update
fedpkg update --type enhancement --notes "Update to 1.2.4"
```

## Challenges I've Encountered

### Dependency Hell

Sometimes packages have circular dependencies or version conflicts:

```bash
# Use dnf to understand dependencies
dnf repoquery --requires mypackage
dnf repoquery --whatrequires mypackage

# Check for conflicts
dnf repoclosure
```

### Failed Builds

Common reasons for build failures:

1. **Missing BuildRequires**: Check build.log carefully
2. **Architecture-specific issues**: Test on all supported arches
3. **Test failures**: Sometimes tests need patching for the Fedora environment

### Security Updates

When a CVE is discovered:

```bash
# Mark as security update
fedpkg update --type security \
              --notes "Fix CVE-2025-XXXXX" \
              --bugs 123456

# Request faster karma
# Security updates get pushed faster
```

## Tools That Make Life Easier

1. **rpmdevtools**: Collection of RPM build utilities
2. **mock**: Clean room building
3. **fedora-review**: Automated package review
4. **anitya**: Upstream release monitoring
5. **the-new-hotness**: Automatic update notifications

## Best Practices

1. **Test thoroughly**: Don't assume it works
2. **Read the guidelines**: Fedora packaging guidelines are comprehensive
3. **Communicate**: Use bugzilla, mailing lists, IRC
4. **Be patient**: Reviews take time
5. **Give back**: Review other packages when you can

## Resources

- [Fedora Packaging Guidelines](https://docs.fedoraproject.org/en-US/packaging-guidelines/)
- [How to create an RPM package](https://docs.fedoraproject.org/en-US/quick-docs/creating-rpm-packages/)
- [Join the Package Maintainers](https://fedoraproject.org/wiki/Join_the_package_collection_maintainers)

## Conclusion

Package maintenance might seem daunting at first, but it's incredibly rewarding. You learn a ton about how Linux distributions work, and you're contributing to the open-source ecosystem.

If you're interested in getting started, pick a small package you use and care about, and dive in!

---

*Interested in Fedora packaging? Have questions? Reach out on [GitHub](https://github.com/davdunc)!*
