---
layout: post
title: "tmux를 활용한 Claude Code 사용법 - 터미널 멀티플렉서로 AI 코딩 극대화하기"
date: 2026-03-19 15:30:00 +0900
categories: [Dev Tools]
tags: [tmux, claude-code, productivity, terminal, ai-coding]
---

## 개요

Claude Code는 터미널에서 동작하는 AI 코딩 에이전트다. 그런데 하나의 터미널 세션에서 하나의 에이전트만 돌리고 있다면, Claude Code의 잠재력을 절반도 쓰지 못하고 있는 셈이다. tmux와 결합하면 여러 에이전트를 병렬로 실행하고, 세션을 영구 유지하며, 원격에서도 작업을 이어갈 수 있다. 이 글에서는 tmux로 Claude Code 활용도를 극대화하는 다섯 가지 접근법을 정리한다.

## 1. Agent Teams: 공식 지원되는 멀티 에이전트 협업

Anthropic은 공식 문서에서 Claude Code의 **Agent Teams** 기능을 안내하고 있다. 핵심은 `teammateMode` 설정을 `tmux`로 지정하는 것이다. 이렇게 하면 각 에이전트가 tmux의 별도 pane에서 실행되어, 한 화면에서 여러 에이전트의 작업 상황을 실시간으로 모니터링할 수 있다.

예를 들어 한 에이전트는 백엔드 API를 구현하고, 다른 에이전트는 프론트엔드 컴포넌트를 작성하며, 세 번째 에이전트는 테스트 코드를 생성하는 식이다. split-pane 모드로 시각적으로 배치하면 각 에이전트의 진행 상황을 한눈에 파악할 수 있어 오케스트레이션이 훨씬 수월해진다. 단순히 여러 터미널 탭을 여는 것과는 차원이 다른 구조화된 협업이 가능하다.

## 2. 영구 세션으로 작업 맥락 유지하기

[devas.life의 가이드](https://www.devas.life/how-to-run-claude-code-in-a-tmux-popup-window-with-persistent-sessions/)에서 소개하는 방법은 실용적이면서도 우아하다. 작업 디렉토리를 MD5 해싱하여 폴더별로 고유한 tmux 세션을 자동 생성하고, `display-popup`과 `attach-session`을 결합해 팝업 창 형태로 Claude Code를 띄운다.

이 접근법의 장점은 프로젝트마다 독립된 세션이 유지된다는 것이다. A 프로젝트에서 작업하다가 B 프로젝트로 전환한 뒤 다시 A로 돌아와도, Claude Code가 이전 대화 맥락을 그대로 갖고 있다. 팝업 형태이므로 필요할 때만 불러오고 닫을 수 있어 화면 공간도 효율적으로 사용할 수 있다. 여러 프로젝트를 동시에 관리하는 개발자에게 특히 유용하다.

## 3. VPS + tmux로 어디서든 접속 가능한 AI 코딩 환경

로컬 머신에 종속되지 않는 환경이 필요하다면, VPS에서 Claude Code를 tmux와 함께 운용하는 방법이 있다. [Medium의 가이드](https://medium.com/@0xmega/claude-code-on-a-vps-the-complete-setup-security-tmux-mobile-access-2d214f5a0b3b)는 세션 생성부터 detach/attach 워크플로우, 그리고 **Tailscale + Termius** 조합으로 모바일에서도 접근하는 방법까지 다룬다.

이 구성의 진짜 가치는 **연속성**에 있다. 사무실에서 시작한 Claude Code 세션을 퇴근길 스마트폰에서 확인하고, 집에서 노트북으로 이어받을 수 있다. tmux 세션은 SSH 연결이 끊어져도 서버에서 계속 살아있으므로, 장시간 걸리는 리팩토링이나 마이그레이션 작업을 맡겨두고 나중에 결과를 확인하는 것도 가능하다.

## 4. amux: 수십 개 에이전트를 병렬로 무인 실행

개인 프로젝트 수준을 넘어 대규모 병렬 실행이 필요하다면, [amux](https://github.com/mixpeek/amux)를 주목할 만하다. tmux 기반으로 수십 개의 Claude Code 에이전트를 동시에 실행하는 오픈소스 멀티플렉서다.

웹 대시보드로 전체 에이전트 상태를 한눈에 파악할 수 있고, 에이전트가 중단되면 자동으로 재시작하는 자가 치유(self-healing) 감시 기능도 갖추고 있다. REST API를 통한 오케스트레이션도 지원하므로, CI/CD 파이프라인이나 자동화 스크립트와 연동하기에 좋다. 모노레포에서 여러 모듈을 동시에 수정하거나, 대량의 반복적 코드 변환 작업을 처리할 때 위력을 발휘한다.

## 5. claude-tmux: 세션 관리에 특화된 TUI 도구

[claude-tmux](https://github.com/nielsgroen/claude-tmux)는 tmux 내에서 여러 Claude Code 세션을 효율적으로 관리하는 데 초점을 맞춘 도구다. TUI(Terminal User Interface)로 세션 상태를 확인하고, 퍼지 필터링 검색으로 원하는 세션을 빠르게 찾을 수 있다.

특히 **git worktree** 지원이 돋보인다. 브랜치마다 별도의 worktree를 만들고 각각에 Claude Code 세션을 붙이면, 여러 기능 브랜치를 동시에 개발하면서도 서로 간섭 없이 작업할 수 있다. 코드 리뷰를 하면서 동시에 다른 브랜치에서 개발을 진행하는 워크플로우가 자연스럽게 가능해진다.

## 정리

- **Agent Teams**(공식 기능)로 여러 에이전트를 split-pane에 배치하면 구조화된 멀티 에이전트 협업이 가능하다.
- **영구 세션 + 팝업** 패턴으로 프로젝트별 대화 맥락을 유지하면서 화면 공간을 절약할 수 있다.
- **VPS + tmux** 조합은 장소에 구애받지 않는 연속적인 AI 코딩 환경을 제공한다.
- **amux**는 대규모 병렬 에이전트 실행과 자동화에, **claude-tmux**는 세션 관리와 git worktree 기반 브랜치별 작업에 각각 강점이 있다.
- tmux는 단순한 터미널 분할 도구가 아니라, Claude Code의 활용 범위를 근본적으로 확장하는 인프라다.

## 참고 자료

- [Orchestrate teams of Claude Code sessions - Anthropic 공식 문서](https://code.claude.com/docs/en/agent-teams)
- [How to run Claude Code in a Tmux popup window with persistent sessions - devas.life](https://www.devas.life/how-to-run-claude-code-in-a-tmux-popup-window-with-persistent-sessions/)
- [Claude Code On a VPS: The Complete Setup - Medium](https://medium.com/@0xmega/claude-code-on-a-vps-the-complete-setup-security-tmux-mobile-access-2d214f5a0b3b)
- [amux - Claude Code agent multiplexer - GitHub](https://github.com/mixpeek/amux)
- [claude-tmux: Manage Claude Code within tmux - GitHub](https://github.com/nielsgroen/claude-tmux)
