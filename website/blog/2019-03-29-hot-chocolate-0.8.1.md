---
title: GraphQL - Hot Chocolate 0.8.1
author: Michael Staib
authorURL: https://github.com/michaelstaib
authorImageURL: https://avatars1.githubusercontent.com/u/9714350?s=100&v=4
---

Today we release version 8.1 (0.8.1) of _Hot Chocolate_. This is mainly release brings a lot of bugfixes and two new features.

<!--truncate-->

Originally we did not want to create an interim release between version 8 and version 9.

The focus of this release was mainly instrumentation and stitching refinements.

## Instrumentation


## Stitching Refinements

One new feature that is now available in the stitching layer is support of error filters. This means that you can now write error filters like on a local schema and transform or enrich the query errors from the remote schema.

In order to make it easier to use error filters in these errors we have changed the error structure.

First, with version 8.1 we now rewrite the path of the error automatically, that means that you now should see the correct path of a field error and not anymore the remote path.

Second, we changed the remote extension property of an error to be the original remote error. This way you can access all the information of the original error in your error filter.






| [HotChocolate Slack Channel](https://join.slack.com/t/hotchocolategraphql/shared_invite/enQtNTA4NjA0ODYwOTQ0LTBkZjNjZWIzMmNlZjQ5MDQyNDNjMmY3NzYzZjgyYTVmZDU2YjVmNDlhNjNlNTk2ZWRiYzIxMTkwYzA4ODA5Yzg) | [Hot Chocolate Documentation](https://hotchocolate.io) | [Hot Chocolate on GitHub](https://github.com/ChilliCream/hotchocolate) |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ | ---------------------------------------------------------------------- |


[hot chocolate]: https://hotchocolate.io
[hot chocolate source code]: https://github.com/ChilliCream/hotchocolate
