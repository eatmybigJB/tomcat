```python
def lambda_handler(event, context):
    """
    Pre Token Generation Lambda
    This example simply adds a custom claim 'a': 1
    to the access token
    """

    # debug log so we see incoming event
    print("EVENT\n", event)

    # prepare the override structure
    # In V2/3 event, event["response"]["claimsOverrideDetails"] is where we put overrides
    event.setdefault("response", {})
    event["response"].setdefault("claimsOverrideDetails", {})

    # set custom claims to add or override
    # This will add `"a": "1"` into Access Token
    event["response"]["claimsOverrideDetails"]["claimsToAddOrOverride"] = {
        "a": "1"
    }

    print("RESPONSE\n", event["response"])

    return event
