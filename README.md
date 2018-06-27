# gvds - Go Visual Diff Service

A service for generating image diffs for visual regression testing.

## Aims

* __Be performant__: The quicker the feedback loop, the happier everyone is.
* __Be scalable__: Today it's your landing page; Tomorrow it's 20 SPAs with 30 screenshots, each, duplicated across phone, tablet, laptop, and desktop screen sizes.
* __Be modular__: S3 vs Filesystem; GRPc vs Thrift; ImageMagick vs pixelmatch; Containers and Docker vs processes and ipc; Everyone's going to have different techs that suit them, and you should be able to change your mind later with minimal pain.

## Work in progress

Check the `DESIGN.md` doc for the plan.
