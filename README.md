# rhoso-luna-hsm Role

In order to use Thales Luna Network HSMs as a PKCS#11 backend, the Barbican
images need to be customized to include the [Luna HSM Client](https://www.thalesdocs.com/gphsm/luna/7/docs/network/Content/Home_Luna.htm) software.

The purpose of this role is to:
* Generate new images for the barbican-api and barbican-worker containing the
  [Luna Minimal Client](https://www.thalesdocs.com/gphsm/luna/7/docs/network/Content/install/client_install/linux_docker_minimal_extended.htm)
* Upload those images to a private repository for use in a RHOSO deployment.
* Create any required config to be mounted by the barbican images for connecting to the HSMs
  using [NTLS](https://www.thalesdocs.com/gphsm/luna/7/docs/network/Content/admin_partition/connections/connections.htm#NTLS)

We expect some preparatory steps to be completed prior to execution in order for the
role to complete successfully:
* The Luna Minimal Client tarball has been downloaded from Thales
  * The location of the Luna Minimal Client tarball should be made available at luna_minclient_src.
  * The HSM Appliance Server Certificate (server.pem) file is made available at luna_server_cert_src.
    When using more than one HSM appliance, this file will be a concatenation of all the server certificates.
  * The client certificate and key made available at luna_client_cert_src.  The files are expected
    to be of the form "{{ client_name }}.pem" and "{{ client_name }}Key.pem"
  * The Chrystoki.conf is available at chrystoki_conf_src.
* The certs and Chrystoki.conf will be retrieved from the given locations and stored in a secret (luna_data_secret)
* The PIN (password) to log into the HSM partition will be stored in a secret (login_secret)

A minimal (one that takes the defaults) invocation of this role is shown below.  In this case, the Luna Minimal Client
software and required certificates and configuration files are stored locally under /opt/luna.

    ---
    - hosts: localhost
      vars:
        barbican_dest_image_namespace: "{{ your quay.io account name }}"
        luna_minclient_src: "file:///opt/luna/LunaClient-Minimal-10.7.2-16.x86_64.tar"
        luna_client_name: "{{ name used for client certificate }}"
        luna_partition_password: "{{ lookup('env', 'LUNA_PARTITION_PASSWORD') }}"
        kubeconfig_path: "/path/to/.kube/config"
        oc_dir: "/path/to/oc/bin/dir/"
      roles:
        - rhoso_luna_hsm

You can also do the steps separately.

    ---
    - hosts: localhost
      vars:
        barbican_dest_image_namespace: "{{ your quay.io account name }}"
        luna_minclient_src: "file:///opt/luna/LunaClient-Minimal-10.7.2-16.x86_64.tar"
       tasks:
       - name: Create new barbican images with the Luna Minimal Client
         ansible.builtin.include_role:
           name: rhoso_luna_hsm
           tasks_from: create_image

    ---
    - hosts: localhost
      vars:
        luna_client_name: "{{ name used for client certificate }}"
        luna_partition_password: "{{ lookup('env', 'LUNA_PARTITION_PASSWORD') }}"
        kubeconfig_path: "/path/to/.kube/config"
        oc_dir: "/path/to/oc/bin/dir/"
      tasks:
      - name: Create secrets containing NTLS certificates and partition password
        ansible.builtin.include_role:
          name: rhoso_luna_hsm
          tasks_from: create_certs

## Role Variables

### Role Parameters
| Variable      | Type    | Default Value | Description                                                                 |
| ------------- | ------- | ------------- | --------------------------------------------------------------------------- |
| `cleanup`     | boolean | `true`        | Delete staging artifacts created by the role (runs in an `always` block).   |
| `working_dir` | string  | `""` (empty)  | Staging directory. Empty allocates a private tempfile (`0700`). If set, the path must not already exist. |

### Image Generation Variables
| Variable                           | Type    | Default Value                                              | Description                                                                 |
| ---------------------------------- | ------- | ---------------------------------------------------------- | --------------------------------------------------------------------------- |
| `barbican_src_image_registry`      | string  | `quay.io`                                                  | Registry used to pull down the Barbican images                              |
| `barbican_src_image_namespace`     | string  | `podified-antelope-centos9`                                | Registry namespace for the Barbican images                                  |
| `barbican_src_image_tag`           | string  | `current-podified`                                         | Tag used when digests are unset                                             |
| `barbican_src_api_image_name`      | string  | `openstack-barbican-api`                                   | Source image name for the Barbican API                                      |
| `barbican_src_worker_image_name`   | string  | `openstack-barbican-worker`                                | Source image name for the Barbican Worker. Set to the API image name when a single shared image is published (for example S2I). |
| `barbican_src_api_image_digest`    | string  | `""` (empty)                                               | Optional `sha256:<hex>` (or bare hex) to pin the API image                  |
| `barbican_src_worker_image_digest` | string  | `""` (empty)                                               | Optional `sha256:<hex>` (or bare hex) to pin the Worker image               |
| `barbican_dest_image_registry`     | string  | `quay.io`                                                  | Registry used to push the modified images                                   |
| `barbican_dest_image_namespace`    | string  | `podified-antelope-centos9`                                | Registry namespace for the modified images                                  |
| `barbican_dest_image_tag`          | string  | `current-podified-luna`                                    | Tag used to identify the modified images                                    |
| `barbican_dest_api_image_name`     | string  | `openstack-barbican-api`                                   | Destination image name for the modified Barbican API                        |
| `barbican_dest_worker_image_name`  | string  | `openstack-barbican-worker`                                | Destination image name for the modified Barbican Worker                     |
| `luna_minclient_src`               | string  | `file:///opt/luna/LunaClient-Minimal-10.7.2-16.x86_64.tar` | Location of the Luna Minimal Client tarball                                 |
| `image_registry_verify_tls`        | boolean | `true`                                                     | Use TLS verification when pushing/pulling images                            |

### Secret Generation Variables
| Variable                     | Type   | Default Value                             | Description                                                                                   |
| ---------------------------- | ------ | ----------------------------------------- | --------------------------------------------------------------------------------------------- |
| `kubeconfig_path`            | string | None                                      | Full path to kubeconfig file. e.g. `/home/user/.kube/config`                                  |
| `oc_dir`                     | string | None                                      | Full path to the directory containing the `oc` command binary. e.g. `/home/user/.crc/bin/oc/` |
| `luna_client_name`           | string | None                                      | Name of the client certificate.  This must match the certificate and key file names           |
| `luna_partition_password`    | string | None                                      | Password (SO PIN) used to log into the HSM partition                                          |
| `chrystoki_conf_src`         | string | `file:///opt/luna/Chrystoki.conf`         | Full path to the Chrystoki.conf file                                                          |
| `luna_server_cert_src`       | string | `file:///opt/luna/cert/server/server.pem` | Full path to the HSM server certificate                                                       |
| `luna_client_cert_src`       | string | `file:///opt/luna/cert/client`            | Directory path to the directory containing the client certificate and key                     |
| `server_ca_file`             | string | `CAFile.pem`                              | Name to be used for the server certificate once mounted on the container                      |
| `luna_data_secret`           | string | `barbican-luna-data`                      | Name of the secret used to store client and server certificates                               |
| `luna_data_secret_namespace` | string | `openstack`                               | Namespace to be used when creating `luna_data_secret`                                         |
| `login_secret`               | string | `hsm-login`                               | Name of the secret used to store the password to log into the HSM partition                   |
| `login_secret_field`         | string | `PKCS11Pin`                               | Secret key used to store the `luna_partition_password` data in `login_secret`                 |

> Security note: do not store `luna_partition_password` in plaintext vars files.
> Use Ansible Vault or an environment lookup (for example,
> `luna_partition_password: "{{ lookup('env', 'LUNA_PARTITION_PASSWORD') }}"`).
