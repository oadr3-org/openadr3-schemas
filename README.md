# OpenADR 3 enumeration schemas

Enumerations (enums) in OpenADR 3 are used to populate event and report payloads and object attributes, via the valuesMap object. For example, an event may include a payload with valuesMap.type = PRICE and valuesMap.values[0.17], or a resource might contain an attribute where valuesMap.type = LOCATION and values[35.686974, -105.937798].

The advantage of this design is that the protocol can treat enums as application-level content that doesn’t impact the API or its Information Model. The protocol and VTNs are completely disentangled from enums, such that a VTN merely retains and provides any valueMap content. Clients are free to use any of the defined enums or introduce their own at any time with no dependency on a VTN. 

The schemas in this repository provide machine readable definitions of enumerations, including the enumeration label, a description, and constraints. The label is generally used in a valuesMap.type property, and constraints apply to the contents of a corresponding valuesMap.values array. Some enums are used for other purposes, such as currency, units of measure, or other. 

The JSON Schema definitions support automatic code generation of data validation functions for these enumerations.
                                                                                                                                           |
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
