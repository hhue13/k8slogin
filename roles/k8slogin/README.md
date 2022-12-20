Role k8slogin
=============

This role handles login to K8S cluster based on facts being set in the playbook or on envionment variables being set whereby the environment variables have preceedance over the facts being set in the playbook.

This should allow to provide user credentials in a flexible way without having to store then in the playbook or variables files but simply by exporting environment variables before the call.

Requirements
------------

### Ansible collections

This role requires the following collections:

- community.okd.openshift_auth

### Python packages

While testing we've discovered that the following Python packages had to be installed via `pip`:

- requests
- requests-oauthlib

Role Variables
--------------

**Note**: The way how the variables are set / replaced by the role depends on the variables precedence rules of Ansible. If the input variable is set by a `ansible.builtin.set_fact` task before calling the role it can be changed by the role. However if the variable is passed via `vars` to the role it can't be changed as the role params have higher priority.

The following role variables are set and / or will be set by the role via `ansible.builtin.set_fact`:

| Role Var as input            | How to set input                                           | Envionment var | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ---------------------------- | ---------------------------------------------------------- | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| k8slogin_api_url             | use `ansible.builtin.set_fact`                           | K8S_HOST_URL   | The Openshift / Kubernetes API Server URL to be used by the role for login. If the environment variable is set the input variable will be overwritten with the value of the environment variable if `k8slogin_api_url` is set via `ansible.builtin.set_fact`                                                                                                                                                                                                                                          |
| k8slogin_kube_admin_user     | use `ansible.builtin.set_fact`  or set as role varibale | K8S_USER       | User used to login on Openshift / Kubernetes API server on "{{k8slogin_api_url}}". If the environment variable is set the input variable will be overwritten with the value of the environment variable if `k8slogin_kube_admin_user` is set via `ansible.builtin.set_fact`                                                                                                                                                                                                                         |
| k8slogin_kube_admin_password | use `ansible.builtin.set_fact`  or set as role varibale | K8S_PASSWORD   | Password for user {{k8slogin_kube_admin_user}} to login on Openshift / Kubernetes API server on "{{k8slogin_api_url}}". If the environment variable is set the input variable will be overwritten with the value of the environment variable if `k8slogin_kube_admin_password` is set via `ansible.builtin.set_fact`                                                                                                                                                                                |
| k8slogin_api_key             | use `ansible.builtin.set_fact`                           | K8S_API_TOKEN  | This fact will contain the OpenShift / Kubernetes**api_key** to be used by the `kubernetes.core` and `community.okd` collections to work with the Openshift / Kubernetes environment. If the environment variable is set the input variable will be overwritten with the value of the environment variable. If this is set no login will be performed. If the environment variable is set the role var (if set via `ansible.builtin.set_fact`) will be replaced otherwise returned unchanged. |

**NOTE**: The updated `k8slogin_api_url` and `k8slogin_api_key` variables will be needed later in the play for logout. Therefore of these might be overwritten by the respective environment variable don't pass them as a role variable but set them via `ansible.builtin.set_fact` before calling the role.

Example Playbook fragments

Some examples how the role can be used and how variables might be overwritten.

Working playbook 01:

After the invocation of the role the the **k8slogin_api_key** is set and contains the token to work on the cluster

```yaml
    - name: Host-URL is taken from K8S_HOST_URL if set
      ansible.builtin.set_fact:
        k8slogin_api_url: "https://api.cluster.example.com:6443"
    ##
    ## Set login facts
    - name: Set K8S login facts
      ansible.builtin.include_role:
        name: k8slogin
      vars:
        k8slogin_kube_admin_user: "testUser"
        k8slogin_kube_admin_password: "testPwd"
      tags:
        - always

    - name: Debug key & host
      ansible.builtin.debug:
        msg: "k8slogin_api_key={{ k8slogin_api_key }} ; k8slogin_api_url={{ k8slogin_api_url }}"

    - name: If login succeeded, try to log out (revoke access token)
      community.okd.openshift_auth:
        host: "{{ k8slogin_api_url }}"
        state: absent
        api_key: "{{ k8slogin_api_key }}"
        validate_certs: "no"
      when: k8slogin_api_key is defined
      tags:
        - always
```

### Fails as the role variables are not available ('k8slogin_api_url' is undefined)

In this example the variable is undefined as `public: true` is not set:

```yaml
    - name: Host-URL is taken from K8S_HOST_URL if set
      ansible.builtin.set_fact:
        myCluster: "https://api.cluster.example.com:6443"
    ##
    ## Set login facts
    - name: Set K8S login facts
      ansible.builtin.include_role:
        name: k8slogin
      vars:
        k8slogin_kube_admin_user: "testUser"
        k8slogin_kube_admin_password: "testPwd"
        k8slogin_api_url: "{{ myCluster }}"
      tags:
        - always

    - name: Debug key & host
      ansible.builtin.debug:
        msg: "k8slogin_api_key={{ k8slogin_api_key }} ; k8slogin_api_url={{ k8slogin_api_url }}"

    - name: If login succeeded, try to log out (revoke access token)
      community.okd.openshift_auth:
        host: "{{ k8slogin_api_url }}"
        state: absent
        api_key: "{{ k8slogin_api_key }}"
        validate_certs: "no"
      when: k8slogin_api_key is defined
      tags:
        - always
```

---

### Working playbook 02

With `pubic: true` the example works as the role vars are now in scope:

```yaml
    - name: Host-URL is taken from K8S_HOST_URL if set
      ansible.builtin.set_fact:
        myCluster: "https://api.cluster.example.com:6443"
    ##
    ## Set login facts
    - name: Set K8S login facts
      ansible.builtin.include_role:
        name: k8slogin
        public: true
      vars:
        k8slogin_kube_admin_user: "testUser"
        k8slogin_kube_admin_password: "testPwd"
        k8slogin_api_url: "{{ myCluster }}"
      tags:
        - always

    - name: Debug key & host
      ansible.builtin.debug:
        msg: "k8slogin_api_key={{ k8slogin_api_key }} ; k8slogin_api_url={{ k8slogin_api_url }}"

    - name: If login succeeded, try to log out (revoke access token)
      community.okd.openshift_auth:
        host: "{{ k8slogin_api_url }}"
        state: absent
        api_key: "{{ k8slogin_api_key }}"
        validate_certs: "no"
      when: k8slogin_api_key is defined
      tags:
        - always
```

---

### Working playbook 03

In this code fragment everything is controlled via the environment variables:

First export the environment variables:

```bash
export K8S_USER="testUser"
export K8S_PASSWORD="testPwd"
# export K8S_HOST_URL="https://api.cluster.example.com:6443"
```

and then start the playbook:

```yaml
    ##
    ## Set login facts
    - name: Set K8S login facts
      ansible.builtin.include_role:
        name: k8slogin
        public: true
      tags:
        - always

    - name: Debug key & host
      ansible.builtin.debug:
        msg: "k8slogin_api_key={{ k8slogin_api_key }} ; k8slogin_api_url={{ k8slogin_api_url }}"

    - name: If login succeeded, try to log out (revoke access token)
      community.okd.openshift_auth:
        host: "{{ k8slogin_api_url }}"
        state: absent
        api_key: "{{ k8slogin_api_key }}"
        validate_certs: "no"
      when: k8slogin_api_key is defined
      tags:
        - always
```

### Working playbook 03

In this example the user & password are taken from the environment variable and he URL from an ansible variable

First export the environment variables:

```bash
export K8S_USER="testUser"
export K8S_PASSWORD="testPwd"
```

and then start the playbook:

```yaml
    - name: Host-URL is taken from K8S_HOST_URL if set
      ansible.builtin.set_fact:
        k8slogin_api_url: "https://api.cluster.example.com:6443"
    ##
    ## Set login facts
    - name: Set K8S login facts
      ansible.builtin.include_role:
        name: k8slogin
      tags:
        - always

    - name: Debug key & host
      ansible.builtin.debug:
        msg: "k8slogin_api_key={{ k8slogin_api_key }} ; k8slogin_api_url={{ k8slogin_api_url }}"

    - name: If login succeeded, try to log out (revoke access token)
      community.okd.openshift_auth:
        host: "{{ k8slogin_api_url }}"
        state: absent
        api_key: "{{ k8slogin_api_key }}"
        validate_certs: "no"
      when: k8slogin_api_key is defined
      tags:
        - always

```

---

License
-------

BSD

Author Information
------------------

Written by:

Hermann Huebler (mailto:hhuebler@2innovate.at) - https://2innovate.at
