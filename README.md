The OpenADR 3.0 Definitions document contains several tables of Enumerations that list allowed (known) values for certain fields.  The enumeration tables list tags, such as `USAGE` or `SIMPLE`, and a textual semi-formal definition of the format.

Most of the enumerations describe items which can be added to `valuesMap` arrays in various objects.  For example the VEN and Resource objects have an `attributes` array.  Each entry in the array is a `type` field, whose value is to coincide with a name in the enumerations table, and a `values` field, which is an array of values of the type described in the enumeration.

The JSON Schema definitions in this directory are intended to contain formal definitions of the enumerations, and to support automatic code generation of data validation functions for these enumerations.

The enumerations in question are:

| Name                                                  | Access                                                                                                                                                                                                                                                                                                                                                                                 |
|-------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Event interval payloads (`valuesMap`)                 | [event-interval-payloads.schema.yaml](./event-interval-payloads.schema.yaml)                                                                                                                                                                                                                                                                                                           |
| Report payloads (`valuesMap`)                         | [report-payloads.schema.yaml](./report-payloads.schema.yaml)                                                                                                                                                                                                                                                                                                                           |
| Reading Type values for a report payload descriptor   | [reading-types.schema.yaml](./reading-types.schema.yaml)                                                                                                                                                                                                                                                                                                                               |
| Values for the OPERATING_STATE payload                | This enumeration is defined in-line with the definition for the OPERATING_STATE payload in [report-payloads.schema.yaml](./report-payloads.schema.yaml)                                                                                                                                                                                                                                |
| Values for the DATA_QUALITY payload                   | This enumeration is defined in-line with the definition for the DATA_QUALITY payload in [report-payloads.schema.yaml](./report-payloads.schema.yaml)                                                                                                                                                                                                                                   |
| Target types (`valuesMap`)                            | [target-types.schema.yaml](./target-types.schema.yaml)                                                                                                                                                                                                                                                                                                                                 |
| Attributes for VEN and Resource objects (`valuesMap`) | [attributes.schema.yaml](./attributes.schema.yaml)                                                                                                                                                                                                                                                                                                                                     |
| Units values in report payload descriptor             | [units.schema.yaml](./units.schema.yaml)                                                                                                                                                                                                                                                                                                                                               |
| Currency names                                        | Currency names are defined by ISO-4217 (see [ISO](https://www.iso.org/iso-4217-currency-codes.html), and [SIX Group](https://www.six-group.com/en/products-services/financial-information/data-standards.html#scrollTo=isin)).  For each software platform there are libraries supplying these currency names.  Applications should use these libraries for validating currency names. |
| Program attributes (`valuesMap`)                      | [program-attributes.schema.yaml](./program-attributes.schema.yaml)                                                                                                                                                                                                                                                                                                                     |

## Using the schema files in an OpenADR implementation

The schema files are meant to be used in your application such as to automate data validation for `valuesMap` lists.

Typically the schema's will be converted into type definitions and data validation code for the language ecosystem in which your application is implemented.  There are many tools available for this purpose.  Such tools produce source code that can be added to your application.

If your application adds additional enumerations - for a private extension - you would copy the schema files, and modify them to fit your needs.  The modified schema's would then be shared with partners who are using your private extension.

Otherwise your application uses the schema files in the OpenADR repository.

## Schema's for `valuesMap` entries

The OpenADR `valuesMap` is used by several OpenADR objects.  It supports holding a list of values where each item is marked with a `type`, and holds a `values` array whose allowed value depends on the type.

It is important to validate the contents of the `values` array, taking into account the various allowed values.

An example `valuesMap` for payloads in an interval in a report (in YAML format):

```yaml
report:
    # ...
    resources:
        # ...
        intervals:
            # ...
            payloads:
                - type: SIMPLE_LEVEL
                  values: [ 1 ]
                - type: USAGE
                  values: [ 20.33 ]
                - type: IMPORT_RESERVATION_FEE
                  values: [ 42.24 ]
                - type: DATA_QUALITY
                  values: [ 'BAD' ]
```

Validating a `valuesMap` means to examine each entry, and for the schema matching the `type` value to validate the `values` array.

An example JSON schema (in YAML format):

```yaml
$id: "./report-payloads.schema.json"
# ...
definitions:
    # ...
    SIMPLE_LEVEL:
        $id: 'report-payloads.schema.json#/definitions/SIMPLE_LEVEL'
        description: |
            Simple level that a VEN resource is operating at for each Interval. Payload
            value is an integer 0, 1, 2, 3 corresponding to values in SIMPLE events.
        type: array
        minItems: 1
        maxItems: 1
        items:
            type: integer
            minimum: 0
            maximum: 3
    # ...
```

Because the schema is validating an array, the `values` array, it has to check that the array is correct for the type.  For the `SIMPLE_LEVEL` type, the array must contain a single integer whose value is between 0 and 3, which is what the schema describes.

### Values for `$id` URLs in these schema files

In JSON Schema (https://json-schema.org/learn/getting-started-step-by-step) the `$id` value sets the URI for the schema.

The most universal URI value would be similar to this:

```yaml
$id: "https://schema.openadr.org/3.0.2/report-payloads.schema.json"
```

While the original source file is in YAML format, it is expected to be converted to JSON as described later.  The extension `.schema.json` is used to convey this is a schema file.

Since the server required for hosting schema files at this URL does not exist, the example shows a local URL like this:

```yaml
$id: "./report-payloads.schema.json"
```

Next, as a URL the `#` indicator is used in JSON schema to reference elements within the schema object.

For the schema implementation discussed here, each schema definition is identified this way:

```yaml
$id: 'report-payloads.schema.json#/definitions/SIMPLE_LEVEL'
```

This refers to `definitions.SIMPLE_LEVEL` within the object contained in `report-payloads.schema.json`.

### Code example for validating a `valuesMap` using a JSON Schema

In the Node.js/TypeScript universe the AJV package handles data validation using a JSON Schema or JSON Type Definition.  The latter is similar to JSON Schema.  We'll focus on JSON Schema since it is directly usable by OpenAPI specifications.

```js
const __filename = import.meta.filename;
const __dirname = import.meta.dirname;

import path from 'node:path';
import YAML from 'js-yaml';
import Ajv, {JSONSchemaType} from "ajv";
import addFormats from "ajv-formats";

// This initializes an ajv object, while adding
// some formats which are useful for OpenADR
export const ajv = new Ajv.default({
    // strict: true,
    // allowUnionTypes: true,
    validateFormats: true
});
addFormats.default(ajv);

// The Schema can be read in JSON with a similar function
// that would use JSON.parse
export async function readYAMLSchema(fn: string): Promise<any | undefined> {
    const _yaml = await fsp.readFile(fn, 'utf-8');
    try {
        const _schema = YAML.load(_yaml);
        return _schema;
    } catch (err) {
        return undefined;
    }
}

const _schemaReportPayloads = await readYAMLSchema(
    path.join(__dirname, 'report-payloads.schema.yaml')
);
```

This initializes AJV and reads a schema file.

```js
const validators = new Map<string, {
    type: string,
    validator: Ajv.ValidateFunction
}>;

for (const _key in _schemaReportPayloads.definitions) {
    validators.set(_key, {
        type: _key,
        validator: ajv.compile(_schemaReportPayloads.definitions[_key])
    });
};
```

In the schema, there is a `definitions` object.  Each entry in this object has a _key_ corresponding to a `type` value in an OpenADR enumerations table.

The `validators` object maps from a key name to an object where `type` is the key, and `validator` is the validation function.  The `for` loop uses `ajv.compile` to create these validation functions.

Validating a `valuesMap` is then

```js
const vmap = [
    // .. an OpenADR valuesMap array
];


for (const item of vmap) {
    const validator = validators.get(item.type.trim());
    if (!validator(item.values)) {
        // Validation failed
        console.error(`FAILED validation with `, validator.errors);
    }
}
```

The validation function is run on the `values` array, as discussed earlier.

In AJV, when validation fails the validation function has a field named `errors` containing an array describing the validation errors.



## Conversion of YAML to JSON format

The natural format for a JSON Schema format is arguably in JSON.  For this purpose, YAML and JSON are directly equivalent.  Since it is easier to read and edit a YAML file, it is convenient to store the schema source as YAML and then convert to JSON.  Further, many tools which consume JSON Schema also support reading them in YAML format.

A widely available tool supporting YAML to JSON conversion is: https://mikefarah.gitbook.io/yq

The conversion is simply:

```shell
yq FILE.schema.yaml -o json >./FILE.schema.json
```
