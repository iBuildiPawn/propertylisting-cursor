# Real Estate One-Stop Shop with AI Agent

A comprehensive real estate platform that connects property owners with potential buyers/renters, featuring an AI agent integration using n8n for workflow orchestration.

## Features

- Property listing and search
- User authentication and profiles
- Saved properties
- Messaging system
- AI-powered assistant for property inquiries
- Dashboard for property owners and buyers

## Tech Stack

- **Backend**: Rust with Actix-web
- **Frontend**: Svelte with TypeScript
- **Database**: Self-hosted Supabase (PostgreSQL + Auth + Storage + Realtime)
- **AI Agent Orchestration**: n8n
- **Styling**: Tailwind CSS

## Getting Started

### Prerequisites

- Node.js (v14+)
- Rust (latest stable)
- Docker (for Supabase local development)

### Installation

1. Clone the repository
2. Set up the backend:
   ```bash
   cd backend
   cargo build
   ```

3. Set up the frontend:
   ```bash
   cd frontend
   npm install
   ```

4. Start the development servers:
   - Backend: `cargo run`
   - Frontend: `npm run dev`

## Project Structure

- `/backend` - Rust backend API
- `/frontend` - Svelte frontend application
- `/docs` - Documentation and PRD
- `/supabase` - Supabase configuration and migrations

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the LICENSE file for details.