---
title: Publications
cms_exclude: true

view: 4

banner:
  caption: ''
  image: ''

# Only show current work, technical reports, and thesis
sections:
  - block: collection
    content:
      title: Current Research
      filters:
        folders:
          - publication/current-research
          - publication/underwater-autonomy
      count: 0
      order: desc
    design:
      view: card
      columns: '1'
  - block: collection
    content:
      title: Technical Reports
      filters:
        folders:
          - publication/technical-report
      count: 0
      order: desc
    design:
      view: card
      columns: '1'
  - block: collection
    content:
      title: Thesis
      filters:
        folders:
          - publication/thesis
      count: 0
      order: desc
    design:
      view: card
      columns: '1'
---
