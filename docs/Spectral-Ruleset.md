# Spectral Async Functions

### The Ruleset

The Spectral rule using the `validateProperties` custom function.

```yaml
formats:
  - oas3
  - oas2
rules:
  validateProperties:
    description: sample async function
    message: '{{error}}'
    severity: error
    given: '$.paths...*.schema..properties'
    then:
      function: validateProperties
functions:
  - validateProperties

```
### Custom Function

Custom Function written in JS. It uses a json response body that is mocked from `validation-api.v1.yaml` to check if `todos-api-v1.json` has any schema properties that match with "complted" and "nme".

```javascript
module.exports = async function (targetVal) {
  const propertyNames = Object.keys(targetVal);
  if (!propertyNames.length) return;

  // Get list of property names we have not validated yet
  const propertyNamesToValidate = propertyNames.filter(propertyName => !this.cache.has(propertyName))
  if (propertyNamesToValidate.length) {
    const validationPromise = validatePropertyNames(propertyNamesToValidate);

    // Store validation promise for each property name
    propertyNamesToValidate.forEach(propertyName => {
      this.cache.set(propertyName, validationPromise);
    });
  }

  // Resolve all validation promises to receive invalid property name results
  const validationResults = await Promise.all(propertyNames.map(propertyName => {
    return this.cache.get(propertyName);
  }));
  const uniqueValidationResults = [...new Set(validationResults.flat())]

  // Get invalid property names from results
  const invalidPropertyNames = propertyNames.filter(propertyName => {
    return uniqueValidationResults.includes(propertyName);
  });

  if (invalidPropertyNames && invalidPropertyNames.length) {
    // Return error message for each invalid property name
    return invalidPropertyNames.map(propertyName => ({ message: `\`${propertyName}\` is an invalid property name.` }));
  }
};

async function validatePropertyNames(propertyNamesToValidate) {
  // Validate the keys
  const res = await fetch(`http://127.0.0.1:3101/validate_properties?property_names=${propertyNamesToValidate.join(',')}`, { 
    headers: { prefer: 'code=200, example=invalid' } 
  });

  if (res.ok) {
    const result = await res.json();

    return result.results;
  }

  return [];
}
```