### Table of Contents

Note: to learn about the vpp-agent concepts, plugins and configuration refer to the [user guide](https://github.com/ligato/vpp-agent/wiki/user-guide)

- [Merging Pull Requests](#merging-pull-requests)
- [Defining Protobuf Models](#defining-protobuf-models)

## Merging Pull Requests

The following types should be used for merging pull requests:
- Use **Squash and merge** for PRs related robot tests
  This makes the history a bit cleaner by squashing multiple commits into single one and skip creating merge commit. 
- Use **Create a merge commit** for all other PRs

## Defining Protobuf Models

All defined Protobuf models should follow official [Style Guide](https://developers.google.com/protocol-buffers/docs/style).

### Named models

These are models that need to have name field to logically identify them. 

The best example for such model is **interface** model. The interfaces come from VPP with an ID (*sw_if_index*) field. However this interface ID is not part of the model, instead the interface model has **Name** field, which is then referenced with ID via agent's _index map_.

The models should only be named if these conditions are met:
- there can be multiple items with the same values (e.g. ACLs..)
- there is no primary key (e.g. sw_if_index for interfaces)
- there would be too many fields needed to include in the key for unique identification (and tagging is supported)
- there is more to the item than just its value (e.g. interface "goes" somewhere) <= TODO: define this better
- the item is already _logical_ (e.g. bridge domain)