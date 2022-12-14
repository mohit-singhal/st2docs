Datastore
===============================

The goal of the datastore service is to allow users to store common parameters and their values
within |st2| for reuse in the definition of sensors, actions, and rules. The datastore service
stores the data as a key-value pair. They can be get/set using the |st2| CLI or the |st2|
Python client.

From the sensor and action plugins, since they are implemented in Python, the key-value pairs are
accessed from the |st2| Python client. For rule definitions in YAML/JSON, the key-value pairs are
referenced with a specific string substitution syntax and the references are resolved on rule
evaluation.

Key-Value pairs can also have a TTL associated with them, for automatic expiry.

Storing and Retrieving Key-Value Pairs via CLI
----------------------------------------------

Set the value of a key-value pair:

.. code-block:: bash

    st2 key set os_keystone_endpoint http://localhost:5000/v2.0
    st2 key set aws_cfn_endpoint https://cloudformation.us-west-1.amazonaws.com

Get individual key-value pair or list all:

.. code-block:: bash

    # To list first 50 key-value pairs (default)
    st2 key list
    # To list all the key-value pairs in the datastore
    st2 key list -n -1
    # To list all the system and user scoped datstore items (default behavior)
    st2 key list --scope=all
    # To list all the system scoped key-value pairs
    st2 key list --scope=system
    # To list all the key-value pairs scoped to the current user
    st2 key list --scope=user
    # To list all the key-value pairs scoped to a particular user
    # NOTE: When RBAC is enabled, only admins can list key-value pairs scoped to
    # a different user. Regular users can only list key-value pairs scoped to
    # themselves.
    st2 key list --scope=user --user=john

    # Get value for key "os_keystone_endpoint"
    st2 key get os_keystone_endpoint
    # Get value for key "os_keystone_endpoint" in json format
    st2 key get os_keystone_endpoint -j

Update an existing key-value pair:

.. code-block:: bash

    st2 key set os_keystone_endpoint http://localhost:5000/v3

Delete an existing key-value pair:

.. code-block:: bash

    st2 key delete os_keystone_endpoint

Storing and Retrieving Numbers, Objects and Arrays via CLI
----------------------------------------------------------

Currently, all datastore values are stored as ``string``.  If you want to store
a non-string value, you can store a JSON-serialized version, and then
de-serialize it in your action/sensor code (see
:ref:`Referencing Key-Value Pairs in Action Definitions <referencing-key-value-pairs-in-action-definitions>`
for more info on using Key-Values in Actions).

Storing an ``number`` / ``integer``:

.. code-block:: bash

    st2 key set retention_days 7

Storing an ``object`` using a JSON-serialized string representation:

.. code-block:: bash

    st2 key set complex_data '{"name": "Dave Smith", "age": 7, "is_parent": True}'


Storing an ``array`` using a JSON-serialized string representation:

.. code-block:: bash

    st2 key set number_list '[1, 2, 3, 4]'
    st2 key set object_list '[{"name": "Eric Jones"}, {"name": "Bob Seger"}]'

Loading Key-Value Pairs from a File
-----------------------------------

Load a list of key-value pairs from a JSON file. The following is a JSON example using the same
keys from the examples above:

.. code-block:: json

    [
        {
            "name": "os_keystone_endpoint",
            "value": "http://localhost:5000/v2.0"
        },
        {
            "name": "aws_cfn_endpoint",
            "value": "https://cloudformation.us-west-1.amazonaws.com"
        }
    ]

Load this file using this command:

.. code-block:: bash

    st2 key load mydata.json

The load command can also accept a YAML file. The following example is YAML for the same
key-value pairs as the JSON file above:

.. code-block:: yaml

    ---
    - name: os_keystone_endpoint
      value: http://localhost:5000/v2.0
    - name: aws_cfn_endpoint
      value: https://cloudformation.us-west-1.amazonaws.com

Load this file using this command:

.. code-block:: bash

    st2 key load mydata.yaml

The load command also allows you to directly load the output of the ``st2 key list -j`` command.
If you have more than 50 key-value pairs, use ``st2 key list -n -1 -j`` to export all keys. This
is useful if you want to migrate datastore items from a different cluster or if you want to
version control the datastore items and load them from version controlled files:

.. code-block:: bash

    # JSON
    st2 key list -n -1 -j > mydata.json
    st2 key load mydata.json

    # YAML
    st2 key list -n -1 -y > mydata.yaml
    st2 key load mydata.yaml


By default, all values for keys in the file must be strings. However, it is also
possible to set the value to any arbitrary data type supported by JSON/YAML
(hash, array, int, boolean, etc) in the file and have StackStorm convert it to JSON before
loading it into the datastore. To accomplish this, you need to explicity pass the
``-c/--convert`` flag: ``st2 key load -c mydata.json``

Loading non-string content via JSON:

.. code-block:: json

    [
        {
            "name": "managed_hosts",
            "value": [
                {
                    "ip_address": "192.168.1.1",
                    "fqdn": "myhost.domain.tld"
                },
                {
                    "ip_address": "192.168.1.2",
                    "fqdn": "myotherhost.domain.tld"
                }
            ]
        },
        {
            "name": "primary_vlan",
            "value": {
                "tag": 123,
                "note": "General purpose traffic"
            }
        }
    ]

Load this file using this command (values will be converted into JSON strings):

.. code-block:: bash

    $ st2 key load -c mydata.json
    +---------------+-----------------------+--------+--------+------+-----+
    | name          | value                 | secret | scope  | user | ttl |
    +---------------+-----------------------+--------+--------+------+-----+
    | managed_hosts | [{"ip_address":       |        | system |      |     |
    |               | "192.168.1.1",        |        |        |      |     |
    |               | "fqdn":               |        |        |      |     |
    |               | "myhost.domain.tld"}, |        |        |      |     |
    |               | {"ip_address":        |        |        |      |     |
    |               | "192.168.1.2",        |        |        |      |     |
    |               | "fqdn": "myotherhost. |        |        |      |     |
    |               | domain.tld"}]         |        |        |      |     |
    | primary_vlan  | {"note": "General     |        | system |      |     |
    |               | purpose traffic",     |        |        |      |     |
    |               | "tag": 123}           |        |        |      |     |
    +---------------+-----------------------+--------+--------+------+-----+

Loading non-string content via YAML:

.. code-block:: yaml

    ---
    - name: managed_hosts
      value:
          - ip_address: 192.168.1.1
            fqdn: myhost.domain.tld
          - ip_address: 192.168.1.2
            fqdn: myotherhost.domain.tld
    - name: primary_vlan
      value:
          tag: 123
          note: General purpose traffic

Load this file using this command (values will be converted into JSON strings):

.. code-block:: bash

    $ st2 key load -c mydata.yaml
    +---------------+-----------------------+--------+--------+------+-----+
    | name          | value                 | secret | scope  | user | ttl |
    +---------------+-----------------------+--------+--------+------+-----+
    | managed_hosts | [{"ip_address":       |        | system |      |     |
    |               | "192.168.1.1",        |        |        |      |     |
    |               | "fqdn":               |        |        |      |     |
    |               | "myhost.domain.tld"}, |        |        |      |     |
    |               | {"ip_address":        |        |        |      |     |
    |               | "192.168.1.2",        |        |        |      |     |
    |               | "fqdn": "myotherhost. |        |        |      |     |
    |               | domain.tld"}]         |        |        |      |     |
    | primary_vlan  | {"note": "General     |        | system |      |     |
    |               | purpose traffic",     |        |        |      |     |
    |               | "tag": 123}           |        |        |      |     |
    +---------------+-----------------------+--------+--------+------+-----+

.. _datastore-scopes-in-key-value-store:

Scoping Datastore Items
-----------------------

By default, all items in the key-value store are stored in the ``st2kv.system`` scope. This means
every user has access to these variables. Use the Jinja expression ``{{st2kv.system.key_name}}``
to refer to these variables in actions or workflows. Prior to v2.0.1, the scope was called
``system`` and therefore the Jinja expression was ``{{system.key_name}}``. As of v2.2, this is no
longer supported.

Variables can be scoped to a specific user. With authentication enabled, you can now control who
can read or write into those variables. For example, to set the variable ``date_cmd`` for the
currently authenticated user, use:

.. code-block:: bash

    st2 key set date_cmd "date -u" --scope=user

The name of the user is determined by the ``X-Auth-Token`` or ``St2-Api-Key`` header passed with
the API call. From the API call authentication credentials, |st2| will determine the user, and
assign this variable to that particular user.

To retrieve the key, use:

.. code-block:: bash

    st2 key get date_cmd --scope=user

If you want a variable ``date_cmd`` as a system variable, you can use:

.. code-block:: bash

    st2 key set date_cmd "date +%s" --scope=system

or simply:

.. code-block:: bash

    st2 key set date_cmd "date +%s"

This variable won't clash with user variables with the same name. Also, you can refer to user
variables in actions or workflows. The Jinja syntax to do so is ``{{st2kv.user.date_cmd}}``.

Note that the notion of ``st2kv.user`` is available only when actions or workflows are run
manually. The notion of ``st2kv.user`` is non-existent when actions or workflows are kicked off
via rules. So the use of user scoped variables is limited to manual execution of actions or
workflows.

Scope can be set in a JSON/YAML key file by adding the ``scope`` property:

JSON

.. code-block:: json

    [
        {
            "name": "date_cmd",
            "value": "date -u",
            "scope": "user"
        }
    ]

YAML

.. code-block:: yaml

    ---
    - name: date_cmd
      value: date -u
      scope: user

.. _datastore-ttl:

Setting a Key-Value Pair TTL
----------------------------

By default, items do not have any TTL (Time To Live). They will remain in the datastore until
manually deleted. You can set a TTL with key-value pairs, so they will be automatically deleted on
expiry of the TTL.

The TTL is set in seconds. To set a key-value pair for the next hour, use this:

.. code-block:: bash

    st2 key set date_cmd "date +%s" --ttl=3600

Use-cases for setting a TTL include limiting auto-remediation workflows from running too
frequently. For example, you could set a value with a TTL when a workflow is triggered. If the
workflow is triggered again, it could check if the value is still set, and if so, bypass running
the remediation action.

Some users keep a count of executions in the key-value store to set a maximum number of executions
in a time period.

TTL can be set in a JSON/YAML key file by adding the ``ttl`` property with an integer value:

JSON

.. code-block:: json

    [
        {
            "name": "date_cmd",
            "value": "date -u",
            "ttl": 3600
        }
    ]

YAML

.. code-block:: yaml

    ---
    - name: date_cmd
      value: date -u
      ttl: 3600


Storing and Retrieving via Python Client
----------------------------------------

Create new key-value pairs. The |st2| API endpoint is set either via the Client init (base\_url)
or from the environment variable (ST2\_BASE\_URL). The default ports for the API servers are
assumed:

.. code-block:: python

    >>> from st2client.client import Client
    >>> from st2client.models import KeyValuePair
    >>> client = Client(base_url='http://localhost')
    >>> client.keys.update(KeyValuePair(name='os_keystone_endpoint', value='http://localhost:5000/v2.0'))

Get individual key-value pair or list all:

.. code-block:: python

    >>> keys = client.keys.get_all()
    >>> os_keystone_endpoint = client.keys.get_by_name(name='os_keystone_endpoint')
    >>> os_keystone_endpoint.value
    u'http://localhost:5000/v2.0'

Update an existing key-value pair:

.. code-block:: python

    >>> os_keystone_endpoint = client.keys.get_by_name(name='os_keystone_endpoint')
    >>> os_keystone_endpoint.value = 'http://localhost:5000/v3'
    >>> client.keys.update(os_keystone_endpoint)

Delete an existing key-value pair:

.. code-block:: python

    >>> os_keystone_endpoint = client.keys.get_by_name(name='os_keystone_endpoint')
    >>> client.keys.delete(os_keystone_endpoint)

Create an encrypted key-value pair:

.. code-block:: python

    >>> client.keys.update(KeyValuePair(name='os_keystone_password', value='$uper$ecret!', secret=True))

Get and decrypt an encrypted key-value pair:

.. code-block:: python

    >>> os_keystone_password = client.keys.get_by_name(name='os_keystone_password', decrypt=True)
    >>> os_keystone_password.value
    u'$uper$ecret!'


Get all key-value pairs and decrypt any that are encrypted:

.. code-block:: python

    >>> keys = client.keys.get_all(params={'decrypt': True})
    >>> # or
    >>> keys = client.keys.query(decrypt=True)

Update an existing encrypted key-value pair:

.. code-block:: python

    >>> os_keystone_password = client.keys.get_by_name(name='os_keystone_password')
    >>> os_keystone_password.value = 'New$ecret!'
    >>> print os_keystone_password.secret
    True
    >>> client.keys.update(os_keystone_password)
    >>> client.keys.get_by_name(name='os_keystone_password', decrypt=True)
    <KeyValuePair name=os_keystone_password,value=New$ecret!>

Set the TTL when creating a key-value pair:

.. code-block:: python

    >>> from st2client.client import Client
    >>> from st2client.models import KeyValuePair
    >>> client = Client(base_url='http://localhost')
    >>> client.keys.update(KeyValuePair(name='os_keystone_endpoint', value='http://localhost:5000/v2.0', ttl=600))

.. _referencing-key-value-pairs-in-action-definitions:

Referencing Key-Value Pairs in Action Definitions
-------------------------------------------------

Key-value pairs are referenced via specific string substitution syntax in rules. In general, the
variable for substitution is enclosed with double brackets (i.e. ``{{var1}}``). To refer to a
key-value pair, prefix the name with "st2kv.system", e.g. ``{{st2kv.system.os_keystone_endpoint}}``.

A simple action example:

.. code-block:: bash

    st2 key set error_message "Remediation failure"

.. code-block:: yaml

    ---
    description: Remediates a host.
    enabled: true
    runner_type: orquesta
    entry_point: workflows/remediate.yaml
    name: remediate
    pack: default
    parameters:
      host:
        required: true
        type: string
      error_message:
        type: string
        default: "{{ st2kv.system.error_message }}"


There is also support for retrieving ``integer``, ``number``, ``object`` and ``array``
key-value pairs from the datastore. If the values are stored as JSON-serialized
strings, then the data will be automatically parsed into the datatype defined in
the parameter definition:

.. code-block:: bash

    st2 key set username "stanley"
    st2 key set -e password "$ecret1!"
    st2 key set num_network_adapters 1
    st2 key set vlan_config '{"vlan_100_general_use": {"tag": 100, "subnet": "10.1.1.0/24"}, "vlan_200_dmz": {"tag": 200, "subnet": "10.99.1.0/24"}}'
    st2 key set dns_servers '["10.0.0.10", "10.0.0.11"]'

.. code-block:: yaml

    ---
    description: Provisions a VM
    enabled: true
    runner_type: orquesta
    entry_point: workflows/vm_provision.yaml
    name: vm_provision
    pack: default
    parameters:
      fqdn:
        type: string
        required: true
      username:
        type: string
        default: "{{ st2kv.system.username }}"
      password:
        type: string
        default: "{{ st2kv.system.password | decrypt_kv }}"
        secret: true
      num_network_adapters:
        type: integer
        default: "{{ st2kv.system.num_network_adapters }}"
      vlan:
        type: string
        required: true
      vlan_config:
        type: array
        default: "{{ st2kv.system.vlan_config }}"
      dns_servers:
        type: object
        default: "{{ st2kv.system.dns_servers }}"


Referencing Key-Value Pairs in Rule Definitions
-----------------------------------------------

Similar to Action Definitions above, one can refer to a key-value pair by prefixing
the name with ``st2kv.system``, e.g. ``{{ st2kv.system.os_keystone_endpoint }}``.

An example rule is provided below. Please refer to the :doc:`Rules </rules>` documentation for
rule-related syntax.

.. code-block:: json

    {
        "name": "daily_clean_up_rule",
        "trigger": {
            "name": "st2.timer.daily"
        },
        "enabled": true,
        "action": {
            "name": "daily_clean_up_action",
            "parameters": {
                "os_keystone_endpoint": "{{ st2kv.system.os_keystone_endpoint }}"
            }
        }
    }

.. _admin-setup-for-encrypted-datastore:

Securing Secrets (admin only)
-----------------------------

The key-value store allows users to store encrypted values (secrets). Symmetric encryption
using AES-256 is used to encrypt secrets. The |st2| administrator is responsible for generating the
symmetric key used for encryption/decryption. Note that the |st2| operator and administrator
(or anyone else who has access to the key) can decrypt the encrypted values.

To generate a symmetric crypto key, please run:

.. code-block:: bash

    sudo mkdir -p /etc/st2/keys/
    sudo st2-generate-symmetric-crypto-key --key-path /etc/st2/keys/datastore_key.json

We recommend that the key is placed in a private location such as ``/etc/st2/keys/`` and
permissions are set such that only the |st2| API process owner (usually ``st2``)
can read the file, and only root can write to it.

To make sure only ``st2`` and root can access the file on the box, run:

.. code-block:: bash

    sudo usermod -a -G st2 st2                              # Add user ``st2`` to ``st2`` group
    sudo mkdir -p /etc/st2/keys/
    sudo chown -R st2:st2 /etc/st2/keys/                    # Give user and group ``st2`` ownership for key
    sudo chmod o-r /etc/st2/keys/                           # Revoke read access for others
    sudo chmod o-r /etc/st2/keys/datastore_key.json         # Revoke read access for others

Once the key is generated, |st2| needs to be made aware of the key. To do this, edit the st2
configuration file (``/etc/st2/st2.conf``) and add the following lines:

.. code-block:: ini

    [keyvalue]
    encryption_key_path = /etc/st2/keys/datastore_key.json

Once the config file changes are made, restart |st2|:

.. code-block:: bash

  sudo st2ctl restart

Validate you are able to set an encrypted key-value in the datastore:

.. code-block:: bash

  st2 key set test_key test_value --encrypt

If you see errors like ``"MESSAGE: Crypto key not found"``, something has gone wrong with setting
up the keys.

.. _datastore-storing-secrets-in-key-value-store:

Storing Secrets
---------------

.. note::

    Please note that if an admin has not setup an encryption key, you will not be allowed to save
    secrets in the key-value store. Contact your |st2| admin to setup encryption keys as per the
    section above.

To save a secret in the key-value store:

.. code-block:: bash

    st2 key set api_token SECRET_TOKEN --encrypt

By default, getting a key tagged as secret (via ``--encrypt``) will always return encrypted values
only. To get plain text, please run the command with the ``--decrypt`` flag:

.. code-block:: bash

    st2 key get api_token --decrypt

.. note::

    Keep in mind that ``--decrypt`` flag can either be used by an administrator (administrator is
    able to decrypt every value) and by the user who set that value in case of the user-scoped
    datastore items (i.e. if ``--scope=user`` flag was passed when originally setting the value).

If you are using system scoped (``st2kv.system``) or user scoped (``st2kv.user``) datastore items
to store secrets, you can decrypt them and use as parameter values in rules or actions. This is
supported via Jinja filter ``decrypt_kv`` (read more about :ref:`Jinja filters<applying-filters-with-jinja>`).
For example, to pass a decrypted password as a rule parameter, use:

.. code-block:: YAML

    aws_key: "{{st2kv.system.aws_key | decrypt_kv}}"

.. note::

    When using ``decrypt_kv`` Jinja filter on a default value of an action parameter you should
    also mark that parameter as secret (``secret: true``). If you don't do that, every user who
    has permission to run (execution) that action will be able to view raw unencryted value of
    that datastore item when executing an action.


Secret keys can be loaded from a JSON/YAML key file by adding the ``secret`` property with
a boolean value. The ``secret`` property only controls how the value is *stored* in the datastore,
not how it is saved in or read from the JSON/YAML key file. The value(s) specified in the JSON/YAML
key file should be the cleartext values.

JSON

.. code-block:: json

    [
        {
            "name": "api_token",
            "value": "SECRET_TOKEN",
            "secret": true
        }
    ]

.. note::

    For the above example, ``"encrypted": false`` is the default so it is not explicitly specified,
    and ``SECRET_TOKEN`` is the cleartext value.

YAML

.. code-block:: yaml

    ---
    - name: api_token
      value: SECRET_TOKEN  # cleartext
      secret: true  # will be stored encrypted
      # encrypted: false (default)

If you would like to save encrypted values in the JSON/YAML key file, see the next section.

Storing Pre-Encrypted Secrets
-----------------------------

In more advanced deployments using the Infrastructure as Code philosophy with Configuration
Management tools like Puppet, Chef, Ansible, etc or Kubernetes it can be useful to import
keys into the datastore that are already encrypted. This technique allows for storing
secrets encrypted at rest, in something like Git or a Vault, then import them into
StackStorm without having to decrypt.

Keep in mind that the values need to be encrypted with the same private key which is used
by the StackStorm instance in question.

To save an already encrypted secret in the key-value store, you need to tell StackStorm
that the value you are passing in is already encrypted and should be used as-is.
This can be done using ``--pre-encrypted`` CLI flag.

.. code-block:: bash

    st2 key set api_token XYZ12fsAz310D --pre-encrypted


Pre-encrypted secret keys can be loaded from a JSON/YAML key file out of the box. All of
the values which contain ``encrypted: true`` and ``secret: true`` are considered as
pre-encrypted and treated as such.

JSON

.. code-block:: json

    [
        {
            "name": "api_token",
            "value": "XYZ12fsAz310D",
            "secret": true,
            "encrypted": true
        }
    ]

.. code-block:: bash

    ``XYZ12fsAz310D`` is the encrypted value

YAML

.. code-block:: yaml

    ---
    - name: api_token
      value: XYZ12fsAz310D  # encrypted value
      secret: true  # store in encrypted format
      encrypted: true  # denotes that the value is already encrypted

Security notes
--------------

We wish to discuss security details and be transparent about the implementation and limitations
of the security practices to attract more eyes to it and therefore build better quality into
security implementations. For the key-value store, we have settled on AES-256 symmetric encryption
for simplicity. For StackStorm v2.7 and earlier, we used the Python library
`keyczar <https://github.com/google/keyczar>`_ for doing this. Since StackStorm v2.8, we use the
`Python cryptography library <https://github.com/pyca/cryptography>`_ to implement symmetric
encryption.

We have made a trade-off that the |st2| admin is allowed to decrypt the secrets in the key-value
store. This made our implementation simpler. We are looking into how to let users pass their own
keys for encryption every time they want to consume a secret from the key-value store. This
requires more UX thought and also moves the responsibility of storing keys to the users. Your
ideas are welcome here.

Please note that the global encryption key means that users with direct access to the database
will only see encrypted secrets in the database. Still, the onus is on the |st2| admin to restrict
access to the database via network daemons only and not allow physical access to the box (or run
databases on different boxes to st2). Note that several layers of security need to be in place,
beyond the scope of this document. While we can help people with deployment questions on the
StackStorm Slack community, please follow your own best security practices guide.
