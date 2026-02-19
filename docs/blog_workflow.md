# Blog Workflow (Nabi + Owner)

## 1) Request format
When requesting a post, send:
- Topic
- Target audience (recruiter / engineer / general)
- 3 must-include experiences/facts
- Tone & depth (high / medium / low)
- Length (short / normal / long)

## 2) Draft process
1. Owner sends topic + constraints
2. Nabi creates outline
3. Nabi writes draft in `_drafts/` or `_posts/`
4. Nabi opens PR with summary/checklist
5. Owner reviews and requests edits
6. Merge after approval

## 3) File conventions
- Draft: `_drafts/YYYY-MM-DD-slug.md`
- Published post: `_posts/YYYY-MM-DD-slug.md`
- Use kebab-case slugs

## 4) Front matter template
```yaml
---
layout: post
title: ""
date: 2026-01-01 09:00:00 +0900
categories: [ai, career]
tags: [bio-ai, challenge, paper]
excerpt: ""
---
```

## 5) Quality checklist
- Facts vs opinions are clearly separated
- Quantified results are explicit
- Sensitive/private information excluded
- "Problem → Approach → Validation → Result → Lessons" structure applied
