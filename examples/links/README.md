# Links Examples

These examples need a little context to make sense of how individual links are
connected. A single example link by itself is not too useful in understanding
how links are used, and this folder is to give context to the example to help
describe how links can be used as described in the API.

More information can be found in the [links proposal]()

## Format

The format of these examples will be 0_<event>.json, where `0` represents the
order in which it was sent.  `embedded` signifies whether or not the link is
embedded in the CDEvent context or sent separately.

## Example Use Case

Let us assume we have a source change that just got merged. The [source change
event](https://github.com/cdevents/spec/blob/v0.3.0/source-code-version-control.md#change-merged)
will be consumed by some CI system and kick off the whole chain. We will
describe this example utilizing links and how links can be used to visualize
these connections. For sake of simplicity of this example we will capture only
a few events to illustrate how links can be used separately or within the
CDEvent itself.

These 6 events will utilize links to connect themselves to the appropriate
event or entity:

1. change.merged
2. build.queued
3. build.started
4. build.finished
5. artifact.packaged
6. artifact.published
