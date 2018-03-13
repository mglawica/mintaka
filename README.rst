=======
Mintaka
=======

:Status: Proof of Concept

The basic idea here is to start with Dokku_like experience using the
following tools:

* vagga_
* lithos_
* cantal_
* verwalter_
* ciruela_
* swindon_

.. _Dokku: http://dokku.viewdocs.io/dokku
.. _lithos: https://lithos.readthedocs.io
.. _vagga: https://vagga.readthedocs.io
.. _cantal: https://cantal.readthedocs.io
.. _verwalter: https://verwalter.readthedocs.io
.. _ciruela: https://tailhook.github.io/ciruela
.. _swindon: https://swindon-rs.github.io/swindon


Installation
============

Only ubuntu xenial is currently supported.

Command-line::

    curl https://raw.githubusercontent.com/mglawica/mintaka/master/mintaka-ubuntu-xenial.list | sudo tee /etc/apt/sources.list.d/mintaka.list
    apt-get update && apt-get install -y mintaka=0.1.1+xenial1


License
=======

Licensed under either of

* Apache License, Version 2.0, (./LICENSE-APACHE or http://www.apache.org/licenses/LICENSE-2.0)
* MIT license (./LICENSE-MIT or http://opensource.org/licenses/MIT)

at your option.

------------
Contribution
------------

Unless you explicitly state otherwise, any contribution intentionally
submitted for inclusion in the work by you, as defined in the Apache-2.0
license, shall be dual licensed as above, without any additional terms or
conditions.
