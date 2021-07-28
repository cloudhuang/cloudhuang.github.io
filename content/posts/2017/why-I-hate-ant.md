---
title: Why I hate Ant
date: 2017-10-07
tags: ['Apache Ant']
---

Apache Ant, yes, this is a great Java build tool I know of. It's free (in all senses of the word), it's a defacto standard in the 20th I think, and it generally works.

But to working with the plain ant scripts is really painfully
* Write ant scripts are complex and verbose.
* Ant build files are generally typeless. There is no grand schema or DTD they can validate against.
* Hardly to maintain the ant scripts
* It's almost impossible re-use someone else's Ant target out of the box. Generally, because targets don't take parameters, seems the only way is use the external property
* Ant has limited fault handling rules, and no persistence of state, so it cannot be used as a workflow tool for any workflow other than classic build and test processes.
* It's not design to offer decision or looping structure ( yes, ant-contrib provide little programming function but still very limited )
* Poor IDE support
    * You can't parameterize your targets for them. (For example, specify that your target requires a set of parameters of a specific type)
    * IDEs can't help you edit your build script (For example, code completion or error-checking as you type is very limited).
* You cannot test/debug your ant scripts only if start the full ant build processes`

*(2017-10-07 发布于简书)*