Victims CVE Database [![Database Validation Status](https://travis-ci.org/victims/victims-cve-db.png?branch=master)](https://travis-ci.org/victims/victims-cve-db)
====================

This database contains information regarding CVE(s) that affect various language modules. We currently store version information corresponding to respective modules as understood by select sources.

|Language|Module Type|Metadata|
|--------|-------------|--------|
|Python|PyPi Package|`name`, `version`|
|Java|Maven Artifact|`groupId`, `artifactId`, `version`|

This project is inspired by the great work done by the people at [RubySec](http://rubysec.github.io/).

#### Notes on CVE(s)
If you already have a CVE assigned to your project and would like us to create an entry for it with the correct coordinates, you can send us a [pull request](https://help.github.com/articles/using-pull-requests) or create an issue [here](https://github.com/victims/victims-cve-db/issues) with CVE details and affected components. For more details refer to the [Contributing](#contributing) section of this document.

If you are a project owner/maintainer wanting to request for a CVE, a good resource regarding this is available [here](http://people.redhat.com/kseifrie/CVE-OpenSource-Request-HOWTO.html).

Contributing
------------
We are always looking for contributors. If you would like to submit a new entry or make an update to an existing entry, feel free to send a [pull request](https://help.github.com/articles/using-pull-requests) or create an issue [here](https://github.com/victims/victims-cve-db/issues) on GitHub.

Please do refer to the [Database Internals](#database-internals) section when making contributions.

#### Validating changes/submissions
You can validate your commits on your branch or local repository by:
```sh
# Validates all changes *.yaml files in database compared to upstream/master
bash validation/git-change-validate.sh
```
If you just want to validate a `YAML` file, you can run the provided python script:
```sh
# This requires PyYAML, all requirements are listed in validation/requirements.txt
# pip install -r validation/requirements.txt

# The validation script
python validation/validate_yaml.py <language> file1 [file2 ... fileN]
```

Database Internals
------------------
Each CVE entry in the database is stored as a `YAML` file.

### Database File Storage Structure
The following structure is employed to store entries:
```
victims-cve-db/database/<language>/<year>/<cve-id>.yaml
```

As an example, the entry for `CVE-2012-1150` would be:
```
victims-cve-db/database/python/2012/1150.yaml
```
### The YAML Document
The document requires all `required` fields in the [Common Content Schema](#common-content-schema).

#### Common Content Schema
|Field|Requirement|Type|Description|
|-----|-----------|----|-----------|
|`cve`|`required` |`string`:`YYYY-[0-9]*`|The CVE ID identifying the security flaw.|
|`title`|`required`|`string`|The flaw's title or short summary.|
|`description`|`optional`|`text`|Long description of the flaw.|
|`cvss_v2`|`optional`|`float`|The CVSS v2 score for the flaw.|
|`references`|`optional`|`list`:`url`|Reference url(s) for the flaw.|
|`affected`|`required`|`list`:`language-module`|Affected language modules.|
|`hash` | added later | `string` | Hash of the overall package.|
| `file_hashes`| added later | `list`: `dict` of hash and name | Hashes for specific files in the package|
| `package_urls`| required | `list` | List of urls pointing to known vulnerable packages. Used for hashing. |

#### package_url

The package_url field currently supports a templated http string. There will be future supported added for
real [package-urls](https://github.com/package-url/purl-spec). In the meantime the template string supports
a few fields which are generated from the victims-cve-db entry itself:

| Name    | Description                        |
|---------|------------------------------------|
| name    | The name of the artifact           |
| version | The version number of the artifact |

Example ``package_url`` templates include:

```
http://repo.maven.apache.org/maven2/HTTPClient/HTTPClient/{version}/HTTPClient-{version}.jar
```

```
http://repo.spring.io/release/org/aspectj/{name}/{version}/{name}-{version}.jar  
```

This will be expanded when doing package hashing for each affected version in the victims-cve-db entry. As an example if
``http://example.org/{name}-{version}.jar`` had a name of ``test`` a single affected of ``<=2.18.2,2.18`` the hashing
would expand to pull down

- http://example.org/test-2.18.2.jar
- http://example.org/test-2.18.1.jar
- http://example.org/test-2.18.0.jar

You can generate package-urls using a legacy victims-cve-db formatted yaml. The legacy format did not contain package-urls, and only contained version-string. Use the [victims-db-builder](https://github.com/victims/victims-db-builder) project to generate package-urls.

**Note**: Java archives will have their ``name`` pulled from ``artifactId``.

#### `version-string`: `common`
The version strings across all languages are expecte to match the regex:
```re
^(?P<condition>[><=]=)(?P<version>[^, ]+)(?:,(?P<series>[^, ]+)){0,1}$
```
Examples: `<=2.6.1,2.6`, `==2.7.0`, `>=1.0.1_Beta`

Which enforces the format `<condition><version>[,<series>]`. Commas (`,`) and spaces (` `) are considered illegal in `<version>` and `<series>` strings.

The `<series>` string is optional and only used to set boundaries for version ranges. For example, in `<=2.6.1,2.6`, the series is `2.6` and indicates that only versions in the `2.6.x` series with `x<=1` is captured.

#### `language-module`: `python`
|Field|Requirement|Type|Description|
|-----|-----------|----|-----------|
|`name`|`required`|`string`|Affected package name. Use PyPi name where possible.|
|`version`|`required`|`list`:[`version-string`](#version-string-common)|Versions that are vulnerable to this CVE.|
|`fixedin`|`optional`|`list`:[`version-string`](#version-string-common)|Versions that contain a fix for this CVE.|
|`unaffected`|`optional`|`list`:[`version-string`](#version-string-common)|Versions that are not vulnerable to this CVE, this excludes the versions that are in `fixedin`.|

#### `language-module`: `java`
|Field|Requirement|Type|Description|
|-----|-----------|----|-----------|
|`groupId`|`required`|`string`|Maven groupId of the affected artifact.|
|`artifactId`|`required`|`string`|Maven artifactId of the affected artifact.|
|`version`|`required`|`list`:[`version-string`](#version-string-common)|Versions that are vulnerable to this CVE.|
|`fixedin`|`optional`|`list`:[`version-string`](#version-string-common)|Versions that contain a fix for this CVE.|
|`unaffected`|`optional`|`list`:[`version-string`](#version-string-common)|Versions that are not vulnerable to this CVE, this excludes the versions that are in `fixedin`.|
