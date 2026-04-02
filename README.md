# $fillable Has No Context: Why Mass Assignment Breaks Down at Scale

![Why Mass Assignment Breaks Down at Scale](assets/poster.jpg)

## What's Inside

- Why `Model::query()->create($request->validated())` is fine in small projects but breaks down at scale
- The real problem: losing visibility into what each endpoint actually writes to the database
- Security risk when trust boundaries aren't enforced at the model level
- The DTO + Action pattern that makes field mapping explicit and honest
- Real-world example with `OrderCreateDTO` and separate actions per context (customer, admin, frontend)

Read the full post: [$fillable Has No Context: Why Mass Assignment Breaks Down at Scale](https://dev.to/tegos/fillable-has-no-context-why-mass-assignment-breaks-down-at-scale-3lmj)
