## Relation to efficiency

.. example:

    Everyone loves simple code, but reducing complexity takes time. Would you rather have a simple code base that
    has 80% test code coverage, or a complex one that has 95% coverage? If you work on some code that may
    completely change in the coming two weeks, then how much should you invest in testing? If you delay
    testing some of the less important edge-cases until the solution has stabilized, then how will you make sure
    that you remember to do it later? And if you end up with a huge and complex code base, should you try to refactor
    it or rewrite it from scratch?

I won't try to answer the above questions, but I think they are worth asking. Most companies have code bases that
seem way too complex to most people. Invariably, it's clear that refactoring is possible but that it would take a
lot of time. In other words, code simplicity and efficiency are related. However, this relationship is usually
not talked about. I've often heard the complaint from programmers that they don't have enough time to refactor.
Of course, we should expect other teams in the company to support our refactoring efforts, but we should also speak
about how our own trade-offs, efficiency and effectiveness co-determine how much time we can invest in this. I know
it's a provocative and even an unfair argument, but imagine that the sales department says that they are not going to talk to any potential customers for two weeks because they want to reorganize their workflows? What I am trying to
say though is that if we expect other teams to be supportive, which could mean for example
that they have to continue for some more time with using inferior software, then we should
also ask from ourselves to be effective and efficient so that we have more time for refactoring.
