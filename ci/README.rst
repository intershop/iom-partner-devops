This directory contains copies of different versions of IOM blueprint project, which are used to test
`ci-job-template.yml <../ci-job-template.yml>`_, especially its backward compatibility.

.. code-block:: bash

  # example of adding a release version of IOM blueprint project
  git subtree add --prefix ci/blueprint-project/1.3.1 git@github.com:intershop/iom-blueprint-project.git tags/1.3.1 --squash
  git push

  # example of updating a SNAPSHOT version of IOM blueprint project
  git subtree pull --prefix ci/blueprint-project/develop git@github.com:intershop/iom-blueprint-project.git develop --squash
  git push

Each copy of the IOM blueprint project has to be added as an element to parameter *blueprintInstances*
in `azure-pipelines.yml <../azure-pipelines.yml>`_.
