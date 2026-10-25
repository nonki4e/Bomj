// api/profiles.js
import { sql } from '@vercel/postgres';

export default async function handler(req, res) {
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET,POST,OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type');

  if (req.method === 'OPTIONS') {
    return res.status(200).end();
  }

  try {
    if (req.method === 'GET') {
      const id = String(req.query.id || '').trim();

      if (!id) {
        return res.status(400).json({
          ok: false,
          error: 'ID не указан'
        });
      }

      const result = await sql`
        SELECT id, name, level, created_at
        FROM profiles
        WHERE UPPER(id) = UPPER(${id})
        LIMIT 1
      `;

      if (!result.rows.length) {
        return res.status(404).json({
          ok: false,
          error: 'Профиль не найден'
        });
      }

      return res.status(200).json({
        ok: true,
        profile: result.rows[0]
      });
    }

    if (req.method === 'POST') {
      const {
        id,
        name,
        level = 1
      } = req.body || {};

      if (!id) {
        return res.status(400).json({
          ok: false,
          error: 'ID обязателен'
        });
      }

      const normalizedId = String(id)
        .trim()
        .toUpperCase();

      const result = await sql`
        INSERT INTO profiles (id, name, level)
        VALUES (
          ${normalizedId},
          ${String(name || 'Игрок')},
          ${Number(level) || 1}
        )
        ON CONFLICT (id)
        DO UPDATE SET
          name = EXCLUDED.name,
          level = EXCLUDED.level
        RETURNING id, name, level, created_at
      `;

      return res.status(200).json({
        ok: true,
        profile: result.rows[0]
      });
    }

    return res.status(405).json({
      ok: false,
      error: 'Метод не поддерживается'
    });

  } catch (error) {
    console.error(error);

    return res.status(500).json({
      ok: false,
      error: 'Ошибка сервера'
    });
  }
}
