# Document History

1. Does the working group (WG) consensus represent the strong concurrence of a
few individuals, with others being silent, or did it reach broad agreement?

The working group reached broad agreement. The work on this draft was largely
driven by the needs, inputs and feedback of the WebTransport WG members that
attended QUIC WG sessions and/or participated via GitHub.

2. Was there controversy about particular points, or were there decisions where
the consensus was particularly rough?

No.

3. Has anyone threatened an appeal or otherwise indicated extreme discontent? If
so, please summarize the areas of conflict in separate email messages to the
responsible Area Director. (It should be in a separate email because this
questionnaire is publicly available.)

No


4. For protocol documents, are there existing implementations of the contents of
the document? Have a significant number of potential implementers indicated
plans to implement? Are any existing implementations reported somewhere,
either in the document itself (as RFC 7942 recommends) or elsewhere
(where)?

Yes there several independent implementations of the protocol extension,  driven
primarily by WebTransport use cases (including but not limitied to Media over
QUIC). These have not been captured formally. Interoperability hs been reported
in an adhoc nature but has been deemed accurate and true.

# Additional Reviews

5. Do the contents of this document closely interact with technologies in other
IETF working groups or external organizations, and would it therefore benefit
from their review? Have those reviews occurred? If yes, describe which
reviews took place.

Yes. The WebTransport WG requested this extension and have actively
participated in the design, reviews, and implementation.


6. Describe how the document meets any required formal expert review criteria,
such as the MIB Doctor, YANG Doctor, media type, and URI type reviews.

This document defines a new QUIC Transport Parameter (reset_stream_at) and a new
QUIC Frame Type (RESET_STREAM_AT), which require standard IANA allocations under
the QUIC registries and Expert Review per RFC 8126. Martin Thomson, one of the
designted experts for the relevant registries was actively engaged in codepoint
selection of draft -08 in accordance with the registration policy


7. If the document contains a YANG module, has the final version of the module
been checked with any of the recommended validation tools for syntax and
formatting validation? If there are any resulting errors or warnings, what is
the justification for not fixing them at this time? Does the YANG module
comply with the Network Management Datastore Architecture (NMDA) as specified
in RFC 8342?

N/A


8. Describe reviews and automated checks performed to validate sections of the
final version of the document written in a formal language, such as XML code,
BNF rules, MIB definitions, CBOR's CDDL, etc.

N/A


# Document Shepherd Checks

9. Based on the shepherd's review of the document, is it their opinion that this
document is needed, clearly written, complete, correctly designed, and ready
to be handed off to the responsible Area Director?

Yes.


10. Several IETF Areas have assembled lists of common issues that their
reviewers encounter. For which areas have such issues been identified
and addressed? For which does this still need to happen in subsequent
reviews?

N/A


11. What type of RFC publication is being requested on the IETF stream (Best
Current Practice, Proposed Standard, Internet Standard,
Informational, Experimental or Historic)? Why is this the proper type
of RFC? Do all Datatracker state attributes correctly reflect this intent?

The requested publication type is Proposed Standard. This is the proper type
because the document specifies a normative wire-level extension to the core QUIC
transport protocol (RFC 9000), defining new frame types and modifying how
transport-level stream resets behave


12. Have reasonable efforts been made to remind all authors of the intellectual
property rights (IPR) disclosure obligations described in BCP 79? To
the best of your knowledge, have all required disclosures been filed? If
not, explain why. If yes, summarize any relevant discussion, including links
to publicly-available messages when applicable.

The authors have been reminded. There have been no IPR disclosures filed against
this document


13. Has each author, editor, and contributor shown their willingness to be
listed as such? If the total number of authors and editors on the front page
is greater than five, please provide a justification.

Yes

14. Document any remaining I-D nits in this document. Simply running the idnits
tool is not enough; please review the "Content Guidelines" on
authors.ietf.org. (Also note that the current idnits tool generates
some incorrect warnings; a rewrite is underway.)

No true posititve nits.

15. Should any informative references be normative or vice-versa? See the IESG
Statement on Normative and Informative References.

The references are of the correct type.


16. List any normative references that are not freely available to anyone. Did
the community have sufficient access to review any such normative
references?

N/A


17. Are there any normative downward references (see RFC 3967 and BCP
97) that are not already listed in the DOWNREF registry? If so,
list them.

No


18. Are there normative references to documents that are not ready to be
submitted to the IESG for publication or are otherwise in an unclear state?
If so, what is the plan for their completion?

Bo


19. Will publication of this document change the status of any existing RFCs? If
so, does the Datatracker metadata correctly reflect this and are those RFCs
listed on the title page, in the abstract, and discussed in the
introduction? If not, explain why and point to the part of the document
where the relationship of this document to these other RFCs is discussed.

No


20. Describe the document shepherd's review of the IANA considerations section,
especially with regard to its consistency with the body of the document.
Confirm that all aspects of the document requiring IANA assignments are
associated with the appropriate reservations in IANA registries. Confirm
that any referenced IANA registries have been clearly identified. Confirm
that each newly created IANA registry specifies its initial contents,
allocations procedures, and a reasonable name (see RFC 8126).

I have reviewed the section inline with the rules and process guidance for QUIC
extensions described in RFC 9000. The IANA considerations is complete and in
accordance with expectations


21. List any new IANA registries that require Designated Expert Review for
future allocations. Are the instructions to the Designated Expert clear?
Please include suggestions of designated experts, if appropriate.

N/A