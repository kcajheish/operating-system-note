## Security

Challenge of security system
- User’s program can exploit the flaw in the OS to modify resources and data in your OS.
- Flaws are likely to happen when the OS becomes complex and general purpose.

OS provides a secure abstraction for your application.
- e.g. App makes a file unwritable.

We use defensive approaches to achieve security goals.
- confidentiality
    - keep information hidden
    - e.g. your credit card number
- integrity
    - information is not changed by adversary
- authenticity
    - information is created by trusted party
- availability
    - your system isn’t blocked by adversary
- control sharing
    - e.g. only small group of people can access database
- non repudiation
    - people can’t deny what they have done

With a **security policy**, a general secure principle is turned into concrete details.
- e.g. to achieve integrity, user A and B can modify the file, but other users can’t.

Principal
- least common mechanism
    - each user has its own data structure
    - e.g. each process has its own register and page table
- least privilege
    - process may exploit the power when you grant too many privileges
- separate of privilege
    - require more than two parties to perform actions
- open design
    - assume adversary know every details of your secure system
- fail safe default
    - by default make your system secure
- acceptability
    - make the secure system easy to use for everyone
- economy of mechanism
    - keep your system small so it has less flows
- complete mediation
    - every action should be verified whether it meets security policy.

When a process makes a system call, control is given to the OS. OS can identify the process with process id and authorise the process to perform requested service.

You should seek out resources based on your language
- how to turn good design into code
- how to evaluate whether your program meets security goals
